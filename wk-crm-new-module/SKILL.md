---
name: wk-crm-new-module
description: 以最小变更和最大验证创建新 CRM 业务模块。接收需求模板或原型描述作为输入，自动执行完整开发流程（枚举注册→DDL→字段初始化→后端骨架→前端页面→部署验证）。Use when creating a new CRM business module.
license: MIT
compatibility: 需要项目后端（zhongjicheliang/）和前端（dz-zhongji/）代码库。
metadata:
  author: wukong-team
  version: "1.0"
---

创建新 CRM 业务模块的完整开发编排。6 个阶段，每阶段内建验证门禁。

**Input**

用户应提供以下任一输入：
1. **Excel 模板**：填写好的 `references/CRM新模块开发需求模板.xlsx`（推荐，使用 documents 技能解析）
2. **Markdown 模板**：填写好的 `references/CRM新模块开发需求模板.md`
3. **蓝湖原型URL**：`https://lanhuapp.com/...` 分享链接（AI可直接访问截图分析）
4. **文字描述**：模块名称、功能描述、原型截图路径等
5. **口头描述**：如果输入不清晰，使用 AskUserQuestion 工具逐项收集

**蓝湖原型解析**：使用 `browser-use` 技能访问蓝湖URL，截取页面结构、字段、按钮、注释说明。可提取：列表字段、筛选Tab、操作按钮、业务规则等。

**Excel 解析方法**：使用 `documents` 技能读取 Excel 文件，逐 Sheet 解析为结构化参数。Sheet 名对应：模块基础信息→参数卡片、主表字段→DDL、表单字段→wk_crm_field、子表设计→子表DDL、业务规则→Service逻辑、页面与权限→前端配置。

必须收集的参数（缺一不可）：
- 模块中文名 / 英文名（驼峰）
- CrmEnum.type（数字，200-999）
- 菜单基址（wk_admin_menu 目录 ID）
- 主键字段名 / 编号字段名
- 是否需要审批流
- 表单渲染模式（A/B/C，参考 `references/form-rendering-patterns.md`）

**复杂度级别（L1-L4）自动推断规则**：
- L1：无审批 + 无子表
- L2：有审批 + 无子表
- L3：有审批 + 有子表/ERP
- L4：多子表 + 产品配置

**Steps**

## Phase 0：参数收集与冲突检测

1. **收集模块参数**

   如果用户提供了需求模板文件，读取并解析。否则通过提问收集必填参数。复杂度级别（L1-L4）根据"是否有审批""是否有子表"等参数自动推断。

2. **冲突检测**

   读取 `references/module-registry.md` 中的已分配表，执行以下检查，有冲突立即停止并提示用户调整：
   - 查「已分配 CrmEnum.type」表确认 type 数字未被占用
   - 查「已分配菜单基址」表确认基址未被占用
   - （可选）grep `CrmEnum.java` 和 `CrmAuthEnum.java` 做二次验证

3. **产出模块参数卡片**

   汇总为一张表格，用户确认后才能继续。此卡片是后续所有阶段的唯一数据源。

   **门禁**：用户确认参数卡片 ✓

## Phase 1：后端枚举与数据库 DDL

按以下顺序注册（每步标注精确文件路径）：

1. `zhongjicheliang/crm/crm-common/.../constant/CrmEnum.java` — 新增枚举值 + `getMainFieldName()` 新增编号字段映射
2. `zhongjicheliang/examine/examine-web/.../constant/ExamineEnum.java` — 新增审批枚举（仅 L2+，参考 `references/examine-integration.md` §1）
3. `zhongjicheliang/crm/crm-web/.../constant/CrmAuthEnum.java` — getStandardAuthMenuId() 新增 case
4. `zhongjicheliang/crm/crm-web/.../CrmFieldConst.java` — queryInitField() 新增 case
5. `zhongjicheliang/crm/crm-web/.../CrmFieldSortServiceImpl.java` — getDefaultField() 新增 case
6. `zhongjicheliang/crm/crm-web/.../CrmFieldServiceImpl.java` — queryFields() sortMap 新增

DDL 生成：
- 主表 `wk_crm_zj_{tableName}`（含 company_id, batch_id, check_status, deleted）
- 自定义字段数据表（框架自动管理，确认 CrmEnum.getTableName() 正确）
- 子表（L3+，按需求模板 §2.2）

为每个 CREATE TABLE 生成对应 DROP TABLE 回滚 SQL。

**参考实现**：合同变更模块 `CrmJtContractChange`（L3）或 `CrmBidding`（L2）

**门禁**：`mvn -pl crm/crm-web -am -DskipTests compile` 编译通过 ✓

## Phase 2：数据库初始化数据

1. **wk_crm_field INSERT**
   - label = CrmEnum.type
   - field_id 基础值 = 2042500000000000000 + (type * 1000) + 偏移序号
     （每个模块用不同基础值避免冲突，如 type=250 → 基础值=2042500000000250000）
   - company_id 同时生成 0（系统默认）和实际公司 ID 两份
   - type 值参考 `references/module-registry.md` FieldEnum type 数字对照表
   - SELECT 类型 options 存 JSON 数组：`["选项1","选项2"]`

2. **wk_admin_menu INSERT**
   - 目录：parent_id=1（CRM 根菜单，绝不能挂在二级目录下！）
   - 按钮：ID = 基址 + CrmAuthEnum.value（+1~+6），realm 使用标准命名（save/update/index/read/delete/transfer）
   - 标准 6 个按钮：新建(+1) / 编辑(+2) / 查看列表(+3, menu_type=2) / 查看详情(+4) / 删除(+5) / 转移(+6)
   - 非标准按钮（如“作废”）从 +7 开始，menu_type=3

3. **wk_admin_role_menu** — 为管理员角色授权所有新菜单

4. 为每条 INSERT 生成回滚 DELETE

**参考模板**：`references/sql-templates.md`（wk_crm_field / wk_admin_menu / wk_admin_role_menu 标准 SQL）

**门禁**：
- SQL 语法正确 ✓
- field_id < 9223372036854775807 ✓
- 目录 parent_id = 1 ✓

## Phase 3：后端 Service/Controller 骨架

生成以下文件（遵循 `zhongjicheliang/AGENTS.md` 代码规范）：

| 文件 | 说明 |
|------|------|
| `entity/PO/zj/Crm{Module}.java` | 实体类，含标准字段集 |
| `entity/BO/zj/Crm{Module}SaveBO.java` | 保存入参，不暴露 PO |
| `entity/VO/zj/Crm{Module}VO.java` | 查询回显对象 |
| `mapper/zj/Crm{Module}Mapper.java` | Mapper 接口 |
| `mapper/zj/Crm{Module}Mapper.xml` | Mapper XML |
| `service/zj/ICrm{Module}Service.java` | Service 接口 |
| `service/impl/zj/Crm{Module}ServiceImpl.java` | Service 实现骨架 |
| `controller/zj/Crm{Module}Controller.java` | 标准 12 端点 |

标准 Service 方法（按复杂度裁剪）：
- L1：queryField + queryById + addOrUpdate + deleteByIds（4个）
- L2：L1 + changeOwnerUser + 审批提交 + 审批回调（8个，参考 `references/examine-integration.md` §3）
- L3：L2 + 子表保存 + ERP 同步（12个）
- L4：L3 + 多子表 + 产品配置（16个）

**参考实现**：`CrmBiddingServiceImpl.java`（544行，L2 标准模板）

**门禁**：编译通过 + 所有端点 Swagger 可见 ✓

## Phase 4：前端开发

1. **注册层**（必须完成）：
   - `dz-zhongji/src/views/crm/model/crmTypeModel.js` — 模块模型
   - `dz-zhongji/src/router/modules/crm.js` — 路由（permissions 与 realm 一致，title 硬编码中文）
   - `dz-zhongji/src/api/crm/{module}.js` — 标准 CRUD API（13 个函数）

2. **页面层**（三件套，表单渲染模式参考 `references/form-rendering-patterns.md`）：
   - `views/crm/{module}/index.vue` — 列表页
   - `views/crm/{module}/Create.vue` — 新建/编辑页（根据渲染模式 A/B/C 选择不同代码模板）
   - `views/crm/{module}/Detail.vue` — 详情页

3. **列表页头部批量操作工具栏**（index.vue 关键实现）：

   列表页多选后，头部工具栏显示批量操作按钮（如转移/删除）。必须实现以下要素：

   **a) 导入 EventsObj**：
   ```javascript
   import EventsObj from '../model/events'
   ```

   **b) 导入并注册 TransferHandle 组件**（如需转移功能）：
   ```javascript
   import TransferHandle from '@/components/Page/SelectionHandle/TransferHandle'
   // components 中注册：TransferHandle
   ```

   **c) data 中声明转移相关状态**：
   ```javascript
   transferDialogShow: false,
   transferHandleProps: {}
   ```

   **d) computed 中定义 handleOperations**：
   ```javascript
   handleOperations() {
     // ⚠️ 重要：如果模块 realm（如 jtContractChange）与 crmType（如 contractChange）不一致，
     // 不能直接使用 getOperations()，需用正确的 realm key 查权限
     const ops = []
     const authKey = '{realm}'  // 与 wk_admin_menu.realm 一致
     const crmAuth = this.crm?.[authKey]
     if (crmAuth?.transfer) ops.push(EventsObj.transfer)
     if (crmAuth?.delete) ops.push(EventsObj.delete)
     // 如需导出：if (crmAuth?.excelexport) ops.push(EventsObj.export)
     return ops
   }
   ```

   **e) methods 中实现 tableOperationsClick**：
   ```javascript
   tableOperationsClick(type) {
     if (type === 'transfer') {
       this.transferHandleProps = {
         request: crm{Module}TransferAPI,
         params: { ids: this.selectionList.map(item => item[this.rowIdKey]) },
         showRemoveType: true,
         help: this.getHelpObj(this.crmType, 'transfer')
       }
       this.transferDialogShow = true
     }
     // 其他操作类型...
   }
   ```

   **f) template 中添加 TransferHandle 组件**：
   ```html
   <transfer-handle
     v-if="transferDialogShow"
     v-model:dialog-visible="transferDialogShow"
     :props="transferHandleProps"
     @handle="handleHandle({ type: 'transfer' })" />
   ```

   **⚠️ realm vs crmType 权限 key 陷阱**：
   - 后端 `wk_admin_menu.realm` 定义的权限 key（如 `jtContractChange`）原样传递到前端 `this.crm` 对象
   - 前端组件的 `crmType` 可能与 realm 不一致（如 `crmType: 'contractChange'`）
   - `Table.js` mixin 的 `getOperations()` 使用 `this.crm[this.crmType]` 查权限，若 key 不匹配会导致按钮不显示
   - **解决方案**：当 realm ≠ crmType 时，在 `handleOperations` 中直接用正确的 realm key 构造操作列表

   **参考实现**：`contractChange/index.vue`（realm=jtContractChange, crmType=contractChange）

4. **自定义字段注册**（5 个位置，缺一不可）：
   - `CrmFieldServiceImpl.java` queryFields() sortMap — 已在 Phase 1 第 6 步完成
   - `views/admin/crm/customField/index.vue` — label→moduleType 映射 + 图标映射
   - `views/admin/fields/index.vue` — title 映射
   - （可选）`systemFields.js` — 系统字段中文名映射
   - （可选）`isFieldLibDisabledModule` / `initCom()` — 字段库黑名单/字段类型过滤

5. **审批流前端注册**（仅 L2+，9 处，参考 `references/examine-integration.md` §4）：
   - `ExamineInfoSection.vue` — isCRMExamine() label 数组
   - `ExamineInfoSection.vue` — examineClick() → createType 映射
   - `ExamineInfoSection.vue` — examineClick() → crmLabel 映射（独立于 createType，不要遗漏！）
   - `ExamineInfoSection.vue` — getAuthFieldsParams() → mainLabel 映射
   - `AuthFieldsMixin.js` — flowGetCRMDetailRequestFun() ReadAPI
   - `examineApproveParams.js` — EXAMINE_INFO_APPROVE_LABEL_MAP
   - `examineApproveParams.js` — CRM_MESSAGE_APPROVE_LABEL_MAP（未接入待办可跳过）
   - `examineApproveParams.js` — CRM_MESSAGE_CREATE_TYPE_MAP（未接入待办可跳过）
   - `CRMAllCreate.vue` — 组件导入+注册+crmTypeMap 映射

**参考文档**：`references/examine-integration.md` §4（9 处映射清单）+ `doc/悟空经验/CRM新增功能模块完整流程指南.md` §6.2（CRMAllCreate 注册）

**门禁**：前端编译通过 + 路由可访问 ✓

## Phase 5：部署前验证与缓存清理

1. **缓存清理**：
   - Redis 权限缓存：`DEL USER_AUTH_CACHE_KET:{userId}`
   - CRM 字段缓存：`CrmConst.ALL_FIELD_CACHE_NAME`
   - wk_crm_field_sort 旧记录：`DELETE FROM wk_crm_field_sort WHERE label = {type}`

2. **端到端验证清单**（逐项确认）：
   - [ ] 新建功能正常
   - [ ] 编辑功能正常
   - [ ] 删除功能正常
   - [ ] 列表页字段显示正常
   - [ ] 自定义字段设置页能看到模块入口和字段
   - [ ] 自定义字段设置页"编辑"能进入字段设计器
   - [ ] 审批流配置页能看到业务字段（L2+）
   - [ ] 审批人点击"详情"能看到业务数据（L2+）
   - [ ] 字段权限控制生效（审批节点可配置字段读写权限）
   - [ ] ES 索引存在且 mapping 正确（L3+）
   - [ ] 菜单在导航栏正常显示

3. **菜单不显示排查流程**（按优先级，参考 `doc/悟空经验/CRM菜单配置指南.md` §七）：
   1. 浏览器 Console 检查 `JSON.parse(localStorage.getItem('authInfo')).crm.{realm}.index` 是否为 true
   2. 检查 `wk_admin_menu`：目录 parent_id=1，realm 与路由 permissions 一致
   3. 检查 `wk_admin_role_menu`：当前用户角色已授权新菜单
   4. 检查前端路由 `meta.permissions` 与菜单 realm 一致
   5. 清除 Redis 权限缓存（`USER_AUTH_CACHE_KET:{userId}`）
   6. 检查 ES 索引是否存在（`curl ES_HOST:9200/_cat/indices | grep {realm}`）

4. **回滚方案**：汇总 Phase 1-2 所有回滚 SQL，按逆序生成一键回滚脚本

**门禁**：用户逐项确认验证清单 ✓

**Output**

完成后输出：
- 已完成文件清单（按后端/前端/SQL 分类）
- 20+ 注册点完成确认表（标注 ✓/✗）
- 回滚脚本位置
- 后续建议：业务逻辑完善、联调测试、审批流程后台配置

**Guardrails**

- **禁止跳过**：CrmEnum、CrmAuthEnum、wk_crm_field、wk_admin_menu 四个注册点缺一不可
- **一致性约束**：type/realm/label 三处必须一致，引用参数卡片中的值
- **双表一致性**：`wk_crm_field.type` 必须与 `CrmFieldConst` 代码定义一致，否则列表渲染异常
- **CrmHiddenFieldUtil 检查**：确认新模块未被意外列入隐藏字段黑名单
- **不修改无关代码**：只改需要改的，不动其他模块
- **回滚必备**：每个 INSERT/CREATE 都必须有对应回滚 SQL
- **parent_id=1**：菜单目录必须挂在 CRM 根菜单下，绝不能挂在二级目录
- **代码规范**：遵循 `zhongjicheliang/AGENTS.md`，BO/PO/VO 分离，构造注入，@Schema 注解

**Reference Map**

| 文档 | 路径 | 用途 |
|------|------|------|
| **内置引用（references/）** | | |
| 模块注册表 | `references/module-registry.md` | type/基址/FieldEnum 对照表 |
| SQL 初始化模板 | `references/sql-templates.md` | wk_crm_field/menu/role_menu 标准 SQL |
| 审批集成模板 | `references/examine-integration.md` | ExamineEnum/ServiceImpl/前端9处注册 |
| 表单渲染模式 | `references/form-rendering-patterns.md` | 三种表单模式（纯动态/自定义区块/字段级slot） |
| 需求模板(Excel) | `references/CRM新模块开发需求模板.xlsx` | Excel 版需求模板（6个Sheet） |
| 需求模板(MD) | `references/CRM新模块开发需求模板.md` | Markdown 版需求模板 |
| **外部深度参考（按需查阅）** | | |
| 完整流程指南 | `doc/悟空经验/CRM新增功能模块完整流程指南.md` | 10 章完整流程、检查清单 |
| 菜单配置指南 | `doc/悟空经验/CRM菜单配置指南.md` | 权限/菜单注册机制详解 |
| 审批流注册指南 | `doc/悟空经验/CRM审批流字段权限注册指南.md` | 审批映射清单 + CRMAllCreate 注册 |
| 代码规范 | `zhongjicheliang/AGENTS.md` | 编码标准、CRM 实现偏好 |
| L2 参考 | `CrmBiddingServiceImpl.java`（544行） | 标准审批模块模板 |
| L3 参考 | `CrmJtContractChangeServiceImpl.java` | 子表+ERP 模块模板 |
