# WK-CRM Skills

悟空 CRM AI Agent 技能集，为 CRM 模块开发提供标准化、自动化的 AI 辅助工作流。

## 技能列表

| 技能 | 说明 | 阶段 |
|------|------|------|
| [wk-crm-prototype-to-spec](./wk-crm-prototype-to-spec/) | 从原型/描述生成需求文档（MD + XLSX） | 5 Phase |
| [wk-crm-new-module](./wk-crm-new-module/) | 从零创建新 CRM 业务模块 | 6 Phase |
| [wk-crm-audit-module](./wk-crm-audit-module/) | 对已有 CRM 模块进行全面审计与查漏补缺 | 5 Phase |

**开发流水线**：`prototype-to-spec` → `new-module` → `audit-module`

```
蓝湖原型/截图/描述  →  需求文档(MD+XLSX)  →  完整代码  →  审计报告
   /wk-crm-prototype-to-spec    /wk-crm-new-module    /wk-crm-audit-module
```

## wk-crm-prototype-to-spec

根据蓝湖原型或文字描述，填写 CRM 新模块开发需求模板，同步生成 Markdown 和 Excel 双格式文件。输出可直接作为 `/wk-crm-new-module` 的输入。

**5 个阶段**：

1. **Phase 0** — 原型分析与数据提取（蓝湖 URL / 截图 / 文字描述）
2. **Phase 1** — 参数确认与冲突检测（type / 菜单基址 / 复杂度 / 渲染模式）
3. **Phase 2** — 填写 Markdown 模板（7 个章节）
4. **Phase 3** — 生成 Excel 文件（6 个 Sheet，openpyxl）
5. **Phase 4** — 交叉验证（MD 与 XLSX 一致性）

### 参考文档

| 文档 | 用途 |
|------|------|
| `references/excel-sheet-mapping.md` | 6 个 Sheet 与 MD 章节的精确列映射 |
| `references/reference-modules.md` | 已有模块对照表（按复杂度/业务特征选择参考） |
| `scripts/fill_xlsx.py` | Excel 生成 Python 脚本模板 |

## wk-crm-new-module

以**最小变更**和**最大验证**创建新 CRM 业务模块。接收需求模板或原型描述作为输入，自动执行完整开发流程。

**6 个阶段**：

1. **Phase 0** — 参数收集与冲突检测（CrmEnum.type / 菜单基址 / FieldEnum 冲突检查）
2. **Phase 1** — 后端枚举与数据库 DDL（CrmEnum → ExamineEnum → CrmAuthEnum → CrmFieldConst）
3. **Phase 2** — 数据库初始化数据（wk_crm_field / wk_admin_menu / wk_admin_role_menu）
4. **Phase 3** — 后端 Service/Controller 骨架（PO/BO/VO/Mapper/Service/Controller）
5. **Phase 4** — 前端开发（注册层 + 页面层 + 自定义字段 + 审批流 9 处注册）
6. **Phase 5** — 部署前验证与缓存清理

**复杂度级别**：L1（纯 CRUD）→ L2（+审批）→ L3（+子表/ERP）→ L4（多子表+产品配置）

### 参考文档

| 文档 | 用途 |
|------|------|
| `references/module-registry.md` | CrmEnum.type / 菜单基址 / FieldEnum 对照表 |
| `references/sql-templates.md` | wk_crm_field / wk_admin_menu / wk_admin_role_menu 标准 SQL |
| `references/examine-integration.md` | L2+ 审批集成代码模式（ExamineEnum / ServiceImpl / 前端 9 处） |
| `references/form-rendering-patterns.md` | 三种表单渲染模式（A 纯动态 / B 自定义区块 / C 字段级 slot） |
| `references/CRM新模块开发需求模板.xlsx` | Excel 版需求模板（6 个 Sheet） |
| `references/CRM新模块开发需求模板.md` | Markdown 版需求模板 |

## wk-crm-audit-module

对已有 CRM 模块进行全链路审计，支持审计**任何** CRM 模块（无论是否由 wk-crm-new-module 创建）。

**5 个阶段**：

1. **Phase 0** — 模块发现与参数推断（自动扫描代码库提取模块配置）
2. **Phase 1** — 后端枚举与注册完整性检查（7 项：CrmEnum / ExamineEnum / CrmAuthEnum 等）
3. **Phase 2** — 数据库初始化数据检查（wk_crm_field / wk_admin_menu / wk_admin_role_menu / wk_crm_field_sort）
4. **Phase 3** — 后端 Service/Controller 检查（文件完整性 / 方法完整性 / 审批集成 / 代码规范）
5. **Phase 4** — 前端代码检查（注册层 / 页面层 / 自定义字段 / 审批流 9 处）
6. **Phase 5** — 综合审计报告与修复建议（P0-P3 分级 + 一键修复）

### 参考文档

| 文档 | 用途 |
|------|------|
| `references/audit-checklist.md` | 完整的 5 类检查项清单（含 SQL 查询和权威源） |

## 使用方式

这三个技能设计为 AI Agent（如 Claude、Qoder）的 Skill 文件使用：

1. 将技能目录放入项目的 `.claude/skills/` 或对应 AI 工具的技能目录
2. 通过斜杠命令调用：
   - `/wk-crm-prototype-to-spec` — 从原型/描述生成需求文档
   - `/wk-crm-new-module` — 从需求文档生成完整代码
   - `/wk-crm-audit-module` — 对已有模块进行审计
3. 推荐按流水线顺序使用：先 `prototype-to-spec` 生成需求，再 `new-module` 生成代码

## 项目背景

- **项目**：悟空 CRM 定制版（wk_crm）
- **技术栈**：Java 21 + Spring Boot 3.3 + Spring Cloud Alibaba + Vue 3 + Element Plus
- **后端**：`zhongjicheliang/` — 微服务架构（admin / crm / examine / gateway 等）
- **前端**：`dz-zhongji/` — Vue 3 + Vite + TypeScript

## License

MIT
