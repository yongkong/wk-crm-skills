# CRM新增功能模块完整流程指南

## 概述

在悟空CRM系统中新增一个业务功能模块（如合同变更、合同取消等），需要完成以下四个阶段的配置和开发。

---

## 一、数据库表设计

### 1.1 主表设计

```sql
CREATE TABLE `wk_crm_zj_jt_contract_change` (
  `jt_contract_change_id` bigint(20) NOT NULL COMMENT '主键ID',
  `contract_change_num` varchar(64) DEFAULT NULL COMMENT '变更单编号',
  `change_type` varchar(20) DEFAULT NULL COMMENT '变更类型',
  `contract_id` bigint(20) DEFAULT NULL COMMENT '关联合同ID',
  `contract_num` varchar(64) DEFAULT NULL COMMENT '合同编号',
  `kh_name` varchar(100) DEFAULT NULL COMMENT '客户名称',
  `salesperson_name` varchar(50) DEFAULT NULL COMMENT '业务经理',
  `owner_user_id` bigint(20) DEFAULT NULL COMMENT '负责人ID',
  `vin` varchar(500) DEFAULT NULL COMMENT '车辆VIN',
  `car_count` int(11) DEFAULT 0 COMMENT '台数',
  `expected_change_completion_at` date DEFAULT NULL COMMENT '期望完成时间',
  `config_price_diff_total` decimal(12,2) DEFAULT NULL COMMENT '配置差价合计',
  `change_handling_fee_total` decimal(12,2) DEFAULT NULL COMMENT '变更手续费合计',
  `config_diff_and_fee_total` decimal(12,2) DEFAULT NULL COMMENT '单台变更合计',
  `check_status` int(11) DEFAULT 0 COMMENT '审核状态',
  `batch_id` varchar(50) DEFAULT NULL COMMENT '批次ID',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `create_user_id` bigint(20) DEFAULT NULL,
  `company_id` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`jt_contract_change_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='合同变更主表';
```

### 1.2 明细表设计（如有）

```sql
CREATE TABLE `wk_crm_zj_jt_contract_change_content_detail` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `change_id` bigint(20) NOT NULL COMMENT '关联变更单ID',
  `seq_no` int(11) DEFAULT NULL COMMENT '序号',
  `change_detail_code` varchar(20) DEFAULT NULL COMMENT '变更明细代码',
  `change_detail_item` varchar(100) DEFAULT NULL COMMENT '变更明细项',
  `change_detail_desc` varchar(500) DEFAULT NULL COMMENT '变更描述',
  -- 其他明细字段...
  PRIMARY KEY (`id`),
  KEY `idx_change_id` (`change_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='合同变更明细表';
```

### 1.3 注意事项

- 主键ID使用雪花算法生成（Java代码中 `BaseUtil.getNextId()`）
- 必须包含 `company_id` 字段，用于多企业隔离
- 必须包含 `batch_id` 字段，用于关联自定义字段数据
- 必须包含 `check_status` 字段，用于审批状态管理
- 表名前缀 `wk_crm_zj_` 表示中集定制模块

---

## 二、后端开发

### 2.1 枚举定义

**CrmEnum.java** - 业务模块枚举：
```java
JT_CONTRACT_CHANGE(234, "jtContractChange")
```

**ExamineEnum.java** - 审批类型枚举：
```java
CRM_JT_CONTRACT_CHANGE(234, 234, "合同变更", ExamineModuleTypeEnum.Crm)
```

### 2.2 字段配置

**CrmFieldConst.java** - 初始化字段定义：
```java
case JT_CONTRACT_CHANGE:
    filedList.add(new ModelField("changeType", "变更类型", FieldEnum.SELECT));
    filedList.add(new ModelField("contractId", "合同编号", FieldEnum.JT_CONTRACT));
    // ... 其他字段
    break;
```

**CrmFieldSortServiceImpl.java** - 列表字段配置：
```java
case JT_CONTRACT_CHANGE:
    crmFieldList.add(new CrmField("changeType", "变更类型", FieldEnum.SELECT));
    // ... 其他字段
    break;
```

### 2.3 Service实现

必须实现以下接口：
- `queryField(Long id)` - 查询字段配置
- `queryFormPositionField(Long id)` - 查询表单定位字段
- `queryById(Long id)` - 查询详情
- `addOrUpdate(BO bo, boolean isExcel)` - 新增/修改
- `deleteByIds(List<Long> ids)` - 删除

### 2.4 Controller接口

必须包含以下接口：
```java
@PostMapping("/field")      // 查询新增所需字段
@PostMapping("/field/{id}") // 查询修改所需字段
@PostMapping("/add")        // 保存数据
@PostMapping("/update")     // 更新数据
@PostMapping("/delete")     // 删除数据
@PostMapping("/read")       // 查询详情
@PostMapping("/page")       // 分页查询
```

---

## 三、数据库字段初始化

### 3.1 wk_crm_field 表初始化

这是**最关键也最容易遗漏**的一步。前端动态表单依赖此表中的字段记录。

```sql
SET @uid = 0;
SET @base = 2042500000000000000;
SET @cid = 2027277158278090752;  -- 企业ID

-- 变更类型 (SELECT=3)
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id, 
    is_hidden, is_null, operating, sorting, options, create_user_id, update_user_id, create_time, update_time)
VALUES (@base + 1, 'changeType', '变更类型', 3, 1, 234, @cid, 0, 1, 3, 1,
    '["商务变更","配置变更"]', @uid, @uid, NOW(), NOW());

-- 合同编号 (JT_CONTRACT=261)
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id, 
    is_hidden, is_null, operating, sorting, create_user_id, update_user_id, create_time, update_time)
VALUES (@base + 2, 'contractId', '合同编号', 261, 1, 234, @cid, 0, 1, 3, 2,
    @uid, @uid, NOW(), NOW());

-- ... 其他字段
```

### 3.2 字段类型对照表

| FieldEnum | type值 | 说明 |
|-----------|--------|------|
| TEXT | 1 | 单行文本 |
| TEXTAREA | 2 | 多行文本 |
| SELECT | 3 | 单选（options存JSON数组） |
| DATE | 4 | 日期 |
| NUMBER | 5 | 数字 |
| FLOATNUMBER | 6 | 小数（precisions存精度） |
| MOBILE | 7 | 手机 |
| FILE | 8 | 文件/附件 |
| CHECKBOX | 9 | 多选 |
| USER | 10 | 人员 |
| ATTACHMENT | 11 | 附件 |
| STRUCTURE | 12 | 部门 |
| DATETIME | 13 | 日期时间 |
| EMAIL | 14 | 邮箱 |
| ADDRESS | 24 | 地址 |
| WEBSITE | 25 | 网址 |
| PIC | 29 | 图片 |
| SERIAL_NUMBER | 63 | 编号（options存编号规则） |
| FIELD_GROUP | 60 | 字段分组 |
| TAG | 61 | 标签 |
| ATTENTION | 62 | 关注度 |
| DETAIL_TABLE | 45 | 明细表格（子表） |
| AREA_POSITION | 43 | 地址（省市区） |
| CURRENT_POSITION | 44 | 定位 |
| DATE_INTERVAL | 48 | 日期区间 |
| BOOLEAN_VALUE | 41 | 布尔值 |
| PERCENT | 42 | 百分数 |
| HANDWRITING_SIGN | 46 | 手写签名 |
| OPTIONS_TYPE | 49 | 逻辑表单/选项字段 |
| RICH_TEXT_FORMAT | 70 | 富文本 |
| MATRIX_SCALE | 81 | 矩阵量表 |
| SORT | 82 | 排序 |
| DIVIDER | 83 | 分割线 |
| DATA_COLLAPSE | 76 | 折叠分隔线 |
| DATA_UNION | 100 | 数据关联 |
| VIDEO | 74 | 视频 |
| BAR_QR_CODE | 73 | 条码/二维码 |
| SYS_DICT | 115 | 数据字典 |
| JT_CONTRACT | 261 | 关联合同模块（中集定制） |

> **权威来源**：`common/common-web/.../enums/FieldEnum.java`。历史文档中部分 type 值有误，请以源码为准。

### 3.3 注意事项

- `field_id` 是主键，BIGINT有符号，不能超过 `9223372036854775807`
- 推荐使用 `2042500000000000000` 作为基础值，避免与其他模块冲突
- `company_id` 必须与实际企业ID匹配，否则前端查不到字段
- `create_user_id` 和 `update_user_id` 是必填字段，不能为NULL
- SELECT类型的 `options` 字段存储JSON数组格式：`["选项1","选项2"]`
- SERIAL_NUMBER类型的 `options` 存储编号规则JSON

---

## 四、设计表单（自定义字段配置）

### 4.1 让模块出现在"自定义字段设置"列表中

新增模块后，需要在 **5个位置** 注册才能在管理后台的"自定义字段设置"中出现入口：

#### ① 后端 queryFields 的 sortMap

**文件**: `CrmFieldServiceImpl.java` 的 `queryFields()` 方法

```java
// 只有在 sortMap 中注册了的模块才会在前端列表中显示
sortMap.put(CrmEnum.JT_CONTRACT_CHANGE.getType(), 排序权重);
```

#### ② 前端 customField/index.vue 的 label 映射

**文件**: `src/views/admin/crm/customField/index.vue`

```javascript
// handleCustomField 方法中的 moduleType 映射
234: 'crm_jt_contract_change',

// getLableIcon 方法中的图标映射
234: 'wk wk-contract-line',
```

#### ③ 前端 fields/index.vue 的 title 映射

**文件**: `src/views/admin/fields/index.vue`

```javascript
// title 计算属性中的映射
crm_jt_contract_change: '合同变更管理',
```

#### ④ 前端 fields/index.vue 的可选特殊处理

根据模块特性，可能还需要配置：

| 配置项 | 说明 |
|--------|------|
| `isFieldLibDisabledModule` | 不允许从字段库拖入字段的模块黑名单 |
| `showLogic` | 是否启用逻辑表单功能 |
| `initCom()` | 过滤可使用的字段类型 |
| `systemFields.js` | 系统字段中文名映射 |

### 4.2 字段设计器使用

进入"客户管理 > 自定义字段设置 > 对应模块"后，进入字段设计器：

**三栏布局**：
- **左侧**：字段类型库（35+ 种字段，拖拽到中间）
- **中间**：表单画布（二维数组，支持拖拽排序、字段占比调整）
- **右侧**：字段属性配置面板

### 4.3 支持的字段类型

| formType | 名称 | type | 说明 |
|----------|------|------|------|
| text | 单行文本 | 1 | |
| textarea | 多行文本 | 2 | |
| select | 单选 | 3 | 支持逻辑表单 |
| date | 日期 | 4 | |
| number | 数字 | 5 | |
| floatnumber | 小数 | 6 | 可配置精度 |
| mobile | 手机 | 7 | |
| file | 附件 | 8 | |
| checkbox | 多选 | 9 | 支持逻辑表单 |
| user | 人员 | 10 | |
| structure | 部门 | 12 | |
| datetime | 日期时间 | 13 | |
| email | 邮箱 | 14 | |
| website | URL | 25 | |
| pic | 图片 | 29 | |
| desc_text | 描述文字 | 50 | |
| serial_number | 自定义编号 | 63 | 配置编号规则 |
| field_group | 字段分组 | 60 | |
| field_tag | 标签 | 61 | |
| field_attention | 关注度 | 62 | |
| detail_table | 明细表格 | 45 | 子表 |
| position | 地址 | 43 | 省市区 |
| location | 定位 | 44 | |
| date_interval | 日期区间 | 48 | |
| boolean_value | 布尔值 | 41 | |
| percent | 百分数 | 42 | |
| handwriting_sign | 手写签名 | 46 | |
| rich_text_format | 富文本 | 70 | |
| matrix_scale | 矩阵量表 | 81 | |
| sort | 排序 | 82 | |
| divider | 分割线 | 83 | |
| data_union | 数据关联 | 100 | |
| bar_qr_code | 条码/二维码 | 73 | |
| video | 视频 | 74 | |
| sysDict | 数据字典 | 115 | |
| data_collapse | 折叠分隔线 | 76 | |

### 4.4 字段属性配置（operating 权限体系）

每个字段有一个 `operating` 属性值（整数），转为 9 位二进制后，每一位控制一项权限：

| bit | 权限 | 说明 |
|-----|------|------|
| 0 | nameEdit | 可编辑字段名 |
| 1 | deleteEdit | 可删除字段 |
| 2 | defaultEdit | 可编辑默认值 |
| 3 | percentEdit | 可修改字段占比 |
| 4 | nullEdit | 可设置为必填 |
| 5 | uniqueEdit | 可设置为唯一 |
| 6 | hiddenEdit | 可隐藏字段 |
| 7 | optionsEdit | 可编辑选项 |
| 8 | radioEdit | 可编辑单多选 |

默认值 255（二进制 `111111111`）= 全部允许。

### 4.5 逻辑表单（条件显隐）

仅 `select`、`checkbox` 类型字段可配置。

**功能**：选中某个选项时，显示/隐藏其他指定字段。

**存储方式**：
- `remark` 设为 `'options_type'`
- `optionsData` 存入 JSON：`{ "选项名": [fieldAssistId1, fieldAssistId2, ...] }`

### 4.6 表单布局规则

- 每行最多 4 个字段（各占 25%）
- 字段占比可选：25% / 50% / 75% / 100%
- 支持拖拽排序（上下左右移动）
- 支持复制、删除字段
- 数据存储为二维数组，每个字段有 `formPosition` 坐标（如 `"0,0"`, `"2,1"`）

### 4.7 保存流程

```
[用户拖拽设计表单] → [点击保存]
    ↓
前端校验（字段名非空、不重复、非SQL关键字等）
    ↓
展平二维数组，追加 formPosition 坐标
    ↓
POST /crmField/saveField { data: [...], label: 234 }
    ↓
后端 Redis 分布式锁 → 校验字段数量上限(100) → 对比数据库已有字段
    ↓
删除多余字段（同步ES）→ 新增/更新字段记录 → 保存排序、授权、默认值
```

### 4.8 设计表单的注意事项

1. **系统字段 vs 自定义字段**：系统字段（如 `checkStatus`、`ownerUserId`）在 `CrmFieldConst` 中定义，会自动出现在设计器中但部分属性不可编辑
2. **字段名唯一性**：同一模块内字段名不能重复，且不能是 SQL 关键字
3. **字段数量上限**：每个模块最多 100 个字段，明细表格子字段最多 20 个
4. **关联模块字段**：使用 `data_union`(100) 类型关联其他模块数据
5. **编号规则字段**：`serial_number`(63) 类型的 `options` 存储编号生成规则 JSON
6. **设计完成后**：字段配置会保存到 `wk_crm_field` 表，前端 `filedGetFieldAPI` 即可查询到

---

## 五、前端开发

### 5.1 目录结构

```
src/views/crm/contractChange/
├── Create.vue              # 主动变更创建页
├── Detail.vue              # 详情页
├── index.vue               # 列表页
└── components/
    ├── CreatePassive.vue   # 被动变更创建页
    └── SubTable.vue        # 子表组件
```

### 5.2 表单渲染方式选择

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| 动态渲染 (wk-form-items) | 标准表单字段 | 客户、商机、合同取消 |
| 混合模式 | 基础字段+复杂表格 | 车辆改制申请、合同变更 |
| 硬编码 (el-form-item) | 特殊交互场景 | 不推荐，无法支持字段权限 |

**推荐混合模式**：基础字段用 `wk-form-items`，复杂表格保持硬编码。

### 5.3 crmTypeModel.js 注册

```javascript
{
  type: 234,
  name: i18n.global.t('crm.jtContractChange.className'),
  key: 'contractChange',
  labelKey: 'crmJtContractChange',
  primaryKey: 'jtContractChangeId',
  mainFieldKey: 'contractChangeNum'
}
```

### 5.4 API定义

```javascript
// src/api/crm/jtContractChange.js
export function crmJtContractChangeSaveAPI(data) {
  return request({ url: '/crmJtContractChange/add', method: 'post', data })
}
export function crmJtContractChangeReadAPI(id) {
  return request({ url: `/crmJtContractChange/read/${id}`, method: 'post' })
}
// ... 其他API
```

---

## 六、审批流配置（关键！）

### 6.1 前端审批流注册清单

新增审批模块时，必须在以下 **9 个位置** 完成注册：

| # | 文件 | 位置 | 作用 |
|---|------|------|------|
| 1 | ExamineInfoSection.vue | `isCRMExamine()` | label数组中添加新模块的label |
| 2 | ExamineInfoSection.vue | `examineClick()` | label→createType映射 |
| 3 | ExamineInfoSection.vue | `examineClick()` crmLabel | label→业务label映射 |
| 4 | ExamineInfoSection.vue | `getAuthFieldsParams()` | createType→mainLabel映射 |
| 5 | AuthFieldsMixin.js | `flowGetCRMDetailRequestFun()` | createType→ReadAPI映射 |
| 6 | examineApproveParams.js | `EXAMINE_INFO_APPROVE_LABEL_MAP` | label→业务label |
| 7 | examineApproveParams.js | `CRM_MESSAGE_APPROVE_LABEL_MAP` | 待办model→业务label（如需要） |
| 8 | examineApproveParams.js | `CRM_MESSAGE_CREATE_TYPE_MAP` | 待办model→createType（如需要） |
| 9 | CRMAllCreate.vue | import + components + componentName | 审批弹层加载业务Create组件 |

> **注意**：`crmLabel` 映射在 `examineClick()` 方法内部，与 `createType` 映射是**两个独立的映射对象**，不要遗漏。

### 6.2 CRMAllCreate.vue 注册（第9处）

```javascript
// 1. 导入组件
import ContractChangeCreate from '@/views/crm/contractChange/Create'

// 2. 注册组件
components: { ContractChangeCreate }

// 3. componentName 映射（computed 中）
contractChange: 'ContractChangeCreate'
```

> CRMAllCreate.vue 是审批弹层加载业务表单的核心组件，遗漏会导致审批人点击"详情"无法加载业务数据。

### 6.3 遗漏后果

如果未完成上述注册：
- 审批人无法在审批面板中查看业务数据
- 点击"详情"按钮无响应
- 字段权限配置不生效
- `isCRMExamine` 返回 false，使用简单审批模式

---

## 七、菜单权限配置

### 7.1 wk_admin_menu 表

```sql
-- 目录（parent_id=1, realm=jtContractChange, menu_type=1）
INSERT INTO wk_admin_menu (menu_id, parent_id, menu_name, realm, menu_type, sort, status)
VALUES (4700, 1, '合同变更管理', 'jtContractChange', 1, 17, 1);

-- 标准CRUD按钮（menu_type=3，realm固定命名）
INSERT INTO wk_admin_menu (menu_id, parent_id, menu_name, realm, realm_url, menu_type, sort, status) VALUES
(4701, 4700, '新建', 'save', '/crmJtContractChange/addOrUpdate', 3, 1, 1),
(4702, 4700, '编辑', 'update', '/crmJtContractChange/addOrUpdate', 3, 2, 1),
(4703, 4700, '查看列表', 'index', '/crmJtContractChange/queryPageList', 2, 3, 1),
(4704, 4700, '查看详情', 'read', '/crmJtContractChange/queryById', 3, 4, 1),
(4705, 4700, '删除', 'delete', '/crmJtContractChange/deleteByIds', 3, 5, 1),
(4706, 4700, '转移', 'transfer', '/crmJtContractChange/changeOwnerUser', 3, 6, 1);
```

### 7.2 权限拆分示例

如果需要拆分权限（如主动变更/被动变更分开控制）：

```sql
-- 新增细分权限
INSERT INTO wk_admin_menu VALUES (4709, 4700, '新建主动变更', 'saveActive', NULL, NULL, 3, 9, 1, NULL, NULL, 0);
INSERT INTO wk_admin_menu VALUES (4710, 4700, '新建被动变更', 'savePassive', NULL, NULL, 3, 10, 1, NULL, NULL, 0);

-- 为已有角色赋予新权限
INSERT INTO wk_admin_role_menu (role_id, menu_id, company_id, create_time, ...)
SELECT role_id, 4709, company_id, NOW(), ...
FROM wk_admin_role_menu WHERE menu_id = 4701;
```

---

## 八、检查清单

### 8.1 开发阶段

- [ ] 数据库表设计完成（主表+明细表）
- [ ] 后端枚举定义（CrmEnum、ExamineEnum）
- [ ] 后端字段配置（CrmFieldConst、CrmFieldSortServiceImpl）
- [ ] 后端Service/Controller实现
- [ ] wk_crm_field 字段初始化脚本
- [ ] 前端目录结构创建
- [ ] 前端crmTypeModel.js注册
- [ ] 前端API定义

### 8.2 表单设计阶段

- [ ] 后端 queryFields sortMap 注册
- [ ] 前端 customField/index.vue label 映射
- [ ] 前端 fields/index.vue title 映射
- [ ] 在字段设计器中拖拽设计表单布局
- [ ] 配置字段属性（必填、只读、默认值等）
- [ ] 配置逻辑表单（条件显隐，如需要）
- [ ] 保存表单设计

### 8.3 审批流配置

- [ ] ExamineInfoSection.vue - isCRMExamine() 添加label
- [ ] ExamineInfoSection.vue - examineClick() 添加映射
- [ ] ExamineInfoSection.vue - getAuthFieldsParams() 添加映射
- [ ] AuthFieldsMixin.js - flowGetCRMDetailRequestFun() 添加API
- [ ] examineApproveParams.js - 添加映射
- [ ] CRMAllCreate.vue - 注册组件和映射

### 8.4 菜单权限

- [ ] wk_admin_menu 菜单记录
- [ ] wk_admin_role_menu 角色权限分配

### 8.5 验证测试

- [ ] 新建功能正常
- [ ] 编辑功能正常
- [ ] 删除功能正常
- [ ] 列表页字段显示正常
- [ ] 自定义字段设置页能看到字段
- [ ] 审批流配置页能看到字段
- [ ] 审批人点击"详情"能看到业务数据
- [ ] 字段权限控制生效

---

## 九、常见问题

### 9.1 前端表单为空

**原因**：wk_crm_field 表中没有对应记录

**解决**：
```sql
SELECT field_id, field_name, name, type, label, company_id 
FROM wk_crm_field WHERE label = 234;
-- 如果为空，执行初始化脚本
```

### 9.2 审批人看不到业务数据

**原因**：ExamineInfoSection.vue 中未注册label

**解决**：检查 `isCRMExamine()` 的label数组是否包含新模块的label

### 9.3 field_id 插入失败

**错误**：`Out of range value for column 'field_id'`

**原因**：BIGINT有符号最大值是 `9223372036854775807`

**解决**：使用较小的基础值，如 `2042500000000000000`

### 9.4 字段类型不匹配

**现象**：前端渲染的组件与预期不符

**原因**：wk_crm_field.type 与 CrmFieldConst 中的 FieldEnum 不一致

**解决**：确保数据库type值与FieldEnum的type值一致（参考字段类型对照表）

### 9.5 自定义字段设置中看不到模块

**原因**：未在 sortMap 或前端映射中注册

**解决**：
1. 检查 `CrmFieldServiceImpl.queryFields()` 的 sortMap 是否包含该模块
2. 检查 `customField/index.vue` 的 label→moduleType 映射
3. 检查 `fields/index.vue` 的 title 映射

---

## 十、参考文档

- [CRM审批流字段权限注册指南](./CRM审批流字段权限注册指南.md)
- [CRM菜单配置指南](./CRM菜单配置指南.md)
