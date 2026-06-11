---
name: wk-crm-audit-module
description: 对已有 CRM 业务模块代码进行全面审计、验证和查漏补缺。支持审计任何 CRM 模块（无论是否由 wk-crm-new-module 技能创建）。Use when auditing, verifying, or fixing existing CRM modules.
license: MIT
compatibility: 需要项目后端（zhongjicheliang/）和前端（dz-zhongji/）代码库。
metadata:
  author: wukong-team
  version: "1.0"
---

对已有 CRM 模块进行全链路审计。5 个阶段，每阶段产出检查报告。

**与 wk-crm-new-module 的关系**：
- `wk-crm-new-module`：从零创建新模块（6 Phase）
- `wk-crm-audit`：审计已有模块，发现遗漏和问题（5 Phase）

---

**Input**

用户提供以下任一信息即可启动审计：
1. **模块名**：如"合同变更"、"招投标"（最常用）
2. **模块 type**：如 234、231
3. **模块英文名**：如 contractChange、bidding
4. **文件路径**：如 `CrmJtContractChangeServiceImpl.java`
5. **问题描述**：如"合同变更的审批流不工作"（针对性审计）

**Excel 解析方法**：如果用户提供了需求模板，使用 `documents` 技能读取并作为审计基准。

---

**Steps**

## Phase 0：模块发现与参数推断

**目标**：通过扫描代码库，自动发现模块的完整配置信息。

1. **定位模块文件**

   根据用户输入，搜索以下文件：
   - 后端：`Crm{Module}ServiceImpl.java`、`Crm{Module}Controller.java`、`Crm{Module}.java`（PO）
   - 前端：`views/crm/{module}/Create.vue`、`index.vue`、`Detail.vue`
   - API：`api/crm/{module}.js`

2. **推断模块参数卡片**

   从代码中提取：
   - `CrmEnum.type`：从 `CrmEnum.java` 中 grep 模块名
   - `CrmAuthEnum`：从 `CrmAuthEnum.java` 中 grep 模块名
   - `ExamineEnum`：从 `ExamineEnum.java` 中 grep 模块名（如有）
   - `ExamineLabelUtils`：从 `ExamineLabelUtils.java` 中 grep 常量
   - 菜单基址：从 `wk_admin_menu` 查询 `realm LIKE '%{module}%'`
   - 字段配置：从 `wk_crm_field` 查询 `label = {type}`

3. **产出模块参数卡片**

   汇总为一张表格，用户确认模块身份后继续。

   **门禁**：用户确认模块身份 ✓

---

## Phase 1：后端枚举与注册完整性检查

**检查清单**（逐项验证，标注 ✓/✗/⚠）：

### 1.1 CrmEnum 注册
- [ ] `CrmEnum.java` 中存在模块枚举值
- [ ] `getMainFieldName()` 中有对应 case（如有编号字段）
- [ ] `getTableName()` 返回正确的表名

### 1.2 ExamineEnum 注册（L2+ 必须）
- [ ] `ExamineEnum.java` 中存在审批枚举
- [ ] type 和 relType 相同且等于 CrmEnum.type
- [ ] `ExamineModuleTypeEnum.Crm` 正确

### 1.3 CrmAuthEnum 注册
- [ ] `CrmAuthEnum.java` 中存在枚举
- [ ] `getStandardAuthMenuId()` 中有对应 case

### 1.4 CrmFieldConst 注册
- [ ] `CrmFieldConst.java` 的 `queryInitField()` 中有对应 case
- [ ] 初始化字段列表与需求一致

### 1.5 CrmFieldSortServiceImpl 注册
- [ ] `getDefaultField()` 中有对应 case
- [ ] 默认字段排序与需求一致

### 1.6 CrmFieldServiceImpl 注册
- [ ] `queryFields()` 的 sortMap 中有对应条目
- [ ] 字段排序逻辑正确

### 1.7 ExamineLabelUtils 常量（L2+ 必须）
- [ ] `ExamineLabelUtils.java` 中定义了常量
- [ ] ServiceImpl 中正确引用该常量

**输出**：后端注册检查报告（7 项，每项 ✓/✗/⚠）

---

## Phase 2：数据库初始化数据检查

### 2.1 wk_crm_field 检查
- [ ] 存在 `label = {type}` 的字段记录
- [ ] `company_id = 0`（系统默认）和实际公司 ID 两份记录
- [ ] `field_id` 无冲突（不与其他模块重叠）
- [ ] 字段 type 值正确（对照 `references/module-registry.md` FieldEnum type）
- [ ] SELECT 类型的 options 是合法 JSON 数组
- [ ] 字段数量与需求模板一致

### 2.2 wk_admin_menu 检查
- [ ] 目录记录存在，`parent_id = 1`（CRM 根菜单）
- [ ] `realm` 格式正确：目录 realm = 模块名（如 `jtContractChange`），按钮 realm = 纯操作名（`index`/`save`/`update`/`read`/`delete`/`transfer`，不含模块前缀）
- [ ] 标准 6 个按钮存在（新建/编辑/查看列表/查看详情/删除/转移）
- [ ] 按钮 ID = 基址 + CrmAuthEnum.value
- [ ] 非标准按钮（如"作废"）ID 从 +7 开始

### 2.3 wk_admin_role_menu 检查
- [ ] 管理员角色已授权所有新菜单
- [ ] 授权记录数 = 菜单按钮数

### 2.4 wk_crm_field_sort 检查
- [ ] 存在 `label = {type}` 的排序记录
- [ ] 排序记录与 `CrmFieldSortServiceImpl` 代码一致

**输出**：数据库检查报告（4 类，每类多项 ✓/✗/⚠）

---

## Phase 3：后端 Service/Controller 检查

### 3.1 文件完整性
- [ ] PO 实体类存在，含标准字段（id, companyId, checkStatus, examineRecordId 等）
- [ ] SaveBO 存在，不暴露 PO 内部字段
- [ ] VO 存在，用于查询回显
- [ ] Mapper 接口 + XML 存在
- [ ] Service 接口 + Impl 存在
- [ ] Controller 存在，标准端点完整

### 3.2 Service 方法完整性（按复杂度）
- [ ] L1：queryField + queryById + addOrUpdate + deleteByIds
- [ ] L2：L1 + changeOwnerUser + 审批提交 + 审批回调
- [ ] L3：L2 + 子表保存 + ERP 同步
- [ ] L4：L3 + 多子表 + 产品配置

### 3.3 审批集成检查（L2+）
- [ ] `ExamineService` 已注入
- [ ] `addOrUpdate` 中有审批提交逻辑
- [ ] 编辑场景：调用 `supplementFieldInfo` + `addExamineRecord`
- [ ] 新建场景：先 `save` 再提交审批
- [ ] 审批回调方法存在（更新 checkStatus）

### 3.4 代码规范检查
- [ ] BO/PO/VO 分离，不混用
- [ ] 使用构造注入（非 @Autowired 字段注入）
- [ ] @Schema 注解完整
- [ ] 无硬编码魔法值

**输出**：后端代码检查报告（4 类，每类多项 ✓/✗/⚠）

---

## Phase 4：前端代码检查

### 4.1 注册层检查
- [ ] `crmTypeModel.js` 中有模块定义
- [ ] `router/modules/crm.js` 中有路由配置
- [ ] `api/crm/{module}.js` 中有标准 CRUD API（13 个函数）

### 4.2 页面层检查
- [ ] `index.vue` 存在，列表字段配置正确
- [ ] `Create.vue` 存在，表单渲染模式正确（A/B/C）
- [ ] `Detail.vue` 存在（如需要）

### 4.3 自定义字段注册检查（5 处）
- [ ] `CrmFieldServiceImpl.java` queryFields() sortMap（已在 Phase 1 检查）
- [ ] `views/admin/crm/customField/index.vue` label→moduleType 映射
- [ ] `views/admin/crm/customField/index.vue` 图标映射
- [ ] `views/admin/fields/index.vue` title 映射
- [ ] `systemFields.js` 系统字段中文名映射（可选）

### 4.4 审批流前端注册检查（L2+，9 处）
- [ ] `ExamineInfoSection.vue` isCRMExamine() label 数组
- [ ] `ExamineInfoSection.vue` examineClick() createType 映射
- [ ] `ExamineInfoSection.vue` examineClick() crmLabel 映射
- [ ] `ExamineInfoSection.vue` getAuthFieldsParams() mainLabel 映射
- [ ] `AuthFieldsMixin.js` flowGetCRMDetailRequestFun() ReadAPI
- [ ] `examineApproveParams.js` EXAMINE_INFO_APPROVE_LABEL_MAP
- [ ] `examineApproveParams.js` CRM_MESSAGE_APPROVE_LABEL_MAP（如接入待办）
- [ ] `examineApproveParams.js` CRM_MESSAGE_CREATE_TYPE_MAP（如接入待办）
- [ ] `CRMAllCreate.vue` 组件导入+注册+crmTypeMap 映射

**输出**：前端代码检查报告（4 类，每类多项 ✓/✗/⚠）

---

## Phase 5：综合审计报告与修复建议

### 5.1 生成审计报告

汇总 Phase 1-4 的检查结果，产出：

| 类别 | 检查项数 | ✓ 通过 | ✗ 失败 | ⚠ 警告 |
|------|---------|--------|--------|--------|
| 后端枚举注册 | 7 | ? | ? | ? |
| 数据库初始化 | 4类 | ? | ? | ? |
| 后端代码 | 4类 | ? | ? | ? |
| 前端代码 | 4类 | ? | ? | ? |
| **总计** | **?** | **?** | **?** | **?** |

### 5.2 问题分级

- **P0 阻塞**：模块无法运行（如 CrmEnum 未注册、数据库表不存在）
- **P1 严重**：核心功能缺失（如审批流未集成、菜单不显示）
- **P2 一般**：功能不完整（如缺少某些按钮权限、字段排序异常）
- **P3 建议**：代码规范问题（如未使用构造注入、缺少注释）

### 5.3 修复建议

对每个问题提供：
1. **问题描述**：什么缺失/错误
2. **影响范围**：导致什么功能异常
3. **修复方案**：具体代码/SQL 修改
4. **优先级**：P0/P1/P2/P3

### 5.4 一键修复（可选）

用户确认后，可执行自动修复：
- 补全缺失的枚举注册
- 补全缺失的数据库记录
- 补全缺失的前端注册点
- 修正不一致的配置

**门禁**：用户确认修复方案 ✓

---

**Output**

审计完成后输出：
1. **审计报告**：5 类检查的详细结果（✓/✗/⚠）
2. **问题清单**：按优先级排序的问题列表
3. **修复方案**：每个问题的具体修复代码/SQL
4. **修复确认**：用户确认后执行修复

---

**Guardrails**

- **只读优先**：默认只读取和报告，不修改代码。修改前必须用户确认
- **不删除数据**：只补全缺失，不删除已有数据（除非用户明确要求）
- **最小变更**：修复时只改需要改的，不动其他模块
- **回滚必备**：每个修复操作都提供回滚方案
- **一致性优先**：发现不一致时，以源码（CrmEnum.java 等）为权威源

---

**Reference Map**

| 文档 | 路径 | 用途 |
|------|------|------|
| **内置引用（references/）** | | |
| 审计检查清单 | `references/audit-checklist.md` | 完整的 5 类检查项清单 |
| 模块注册表 | `../wk-crm-new-module/references/module-registry.md` | type/基址/FieldEnum 对照表 |
| SQL 模板 | `../wk-crm-new-module/references/sql-templates.md` | 标准 SQL 模板 |
| 审批集成模板 | `../wk-crm-new-module/references/examine-integration.md` | 审批集成代码模式 |
| **外部深度参考（按需查阅）** | | |
| 完整流程指南 | `doc/悟空经验/CRM新增功能模块完整流程指南.md` | 10 章完整流程 |
| 菜单配置指南 | `doc/悟空经验/CRM菜单配置指南.md` | 权限/菜单注册机制 |
| 审批流注册指南 | `doc/悟空经验/CRM审批流字段权限注册指南.md` | 审批映射清单 |
