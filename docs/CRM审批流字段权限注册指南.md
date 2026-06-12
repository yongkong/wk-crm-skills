# CRM审批流字段权限注册指南

## 背景

在悟空CRM系统中，新增一个需要审批的业务模块时，除了后端枚举注册和前端页面开发外，还需要在**前端审批流组件中完成 label/type 映射注册**，否则审批人将无法在审批面板中查看业务数据。

## 问题案例：合同变更（label=234）

合同取消（label=240）在审批流配置中能看到很多字段，但合同变更（label=234）看不到字段。根因是合同变更在前端审批流组件中有 **7处映射遗漏**。

## 新增审批模块的前端注册清单

当新增一个 CRM 审批模块时，需要在以下位置完成注册（以合同变更 label=234, key='contractChange' 为例）：

### 1. ExamineInfoSection.vue — isCRMExamine()
- **文件**: `src/components/Examine/ExamineInfoSection.vue`
- **位置**: `isCRMExamine` 计算属性中的 label 数组
- **作用**: 决定该模块是否启用 CRM 字段授权审批模式
- **遗漏后果**: `isFieldAuthExamine` 返回 false，审批操作区域使用简单按钮模式，不激活字段授权

### 2. ExamineInfoSection.vue — examineClick()
- **位置**: label → createType 映射对象
- **作用**: 审批人点击"详情"时，根据 label 找到对应的 CRM 模块 key，用于加载业务数据
- **遗漏后果**: 点击"详情"无法加载业务表单

### 3. ExamineInfoSection.vue — getAuthFieldsParams()
- **位置**: mainLabel 映射对象（key → label 数值）
- **作用**: 构建获取授权字段列表的请求参数
- **遗漏后果**: 无法获取字段权限配置

### 4. AuthFieldsMixin.js — flowGetCRMDetailRequestFun()
- **文件**: `src/components/Examine/mixins/AuthFieldsMixin.js`
- **位置**: createType → ReadAPI 映射
- **作用**: 返回各模块的详情读取 API 函数
- **遗漏后果**: 审批弹层无法请求业务数据

### 5. examineApproveParams.js — EXAMINE_INFO_APPROVE_LABEL_MAP
- **文件**: `src/components/Examine/examineApproveParams.js`
- **作用**: 审批详情组件（ExamineInfoSection）使用审批流里的 label，但部分旧 CRM 审批 label 需要转换成授权字段接口能识别的业务 label
- **映射关系**: 审批流 label → 业务 label（如 `1: 6` 表示合同审批流label=1 对应业务label=6）
- **调用方**: `getExamineInfoParamsLabel()` 函数，被 ExamineInfoSection 的审批字段授权流程使用
- **遗漏后果**: 审批操作提交时无法正确传递业务 label，导致授权字段接口返回空

### 6. examineApproveParams.js — CRM_MESSAGE_APPROVE_LABEL_MAP
- **作用**: 待办列表（CRMMessage.vue）使用待办的 model 编号，需要转换为授权字段和审批接口使用的业务 label
- **映射关系**: 待办 model → 业务 label（如 `5: 6` 表示合同待办model=5 对应业务label=6）
- **调用方**: `getCRMMessageParamsLabel()` 函数
- **注意**: 合同变更(234)和合同取消(240)目前未接入待办系统，所以此映射中无对应条目

### 7. examineApproveParams.js — CRM_MESSAGE_CREATE_TYPE_MAP
- **作用**: 待办详情弹窗需要根据 model 找到对应的 CRM 详情模块 key，再用模块主键计算 typeId
- **映射关系**: 待办 model → createType key（如 `5: 'contract'` 表示合同待办对应 contract 模块）
- **调用方**: `getCRMMessageCreateType()` 函数
- **注意**: 同上，合同变更和合同取消目前未接入待办系统

## 表单渲染方式选择

### 动态渲染（wk-form-items）
适用于标准表单字段，支持 30+ 种字段类型，通过后台"自定义字段设置"可配置。

### 硬编码表单（el-form-item）
适用于复杂交互场景（如合同变更的31项变更明细、散发件子表、产品配置弹窗等）。

### 混合模式（推荐用于复杂模块）
基础信息字段用 `wk-form-items` 动态渲染 + slot 自定义特殊字段，复杂表格区块保持硬编码 `el-table`。
参考实现：车辆改制申请（vehicleModificationApply/Create.vue）。

#### 混合模式改造要点

**模板层面**：
```html
<create-sections title="申请内容">
  <el-form ref="crmForm" :model="fieldForm" :rules="fieldRules" ...>
    <wk-form-items
      v-for="(children, index) in fieldList"
      :field-from="fieldForm"
      :field-list="children"
      :ignore-fields="ignoreFields"
      @change="formChange">
      <template #default="{ data }">
        <!-- slot 处理特殊字段 -->
        <crm-relative-cell v-if="data && data.field === 'contractId'" ... />
        <div v-else-if="data && data.field === 'vinList'" ... />
        <el-input v-else-if="data && data.field === 'computedField'" disabled ... />
      </template>
    </wk-form-items>
  </el-form>
</create-sections>
<!-- 复杂表格区块保持硬编码 -->
<create-sections title="变更信息">
  <el-table :data="changeDetailList" ...>...</el-table>
</create-sections>
```

**脚本层面**：
1. `data()` 中用 `fieldList: [], fieldForm: {}, fieldRules: {}` 替代硬编码
2. 定义 `IGNORE_FIELDS`（slot 处理的字段）和 `DISABLED_FIELDS`（只读字段）
3. 添加 `fetchFieldList()` — 支持 `config.getFields`（审批模式）和 `filedGetFieldAPI`（普通模式）两条路径
4. 添加 `processFieldList()` — 处理字段属性（disabled、props、value 等）
5. 添加 `formChange()` / `otherChange()` — 处理字段联动和公式
6. 提交用 `getSubmiteParams(getSubmitFieldList(), fieldForm)` + `decorateSubmitParams()` 补充自定义数据

**审批弹层效果**：改造后，审批人点击"详情"时，`CRMAllCreate` 弹层中的 wk-form-items 会根据 `authLevel` 控制字段显隐和可编辑状态，实现字段权限控制。

#### 重要：数据库字段初始化

从硬编码改为动态表单后，必须确保 `wk_crm_field` 表中有对应记录：

```sql
-- 检查字段是否存在
SELECT field_id, field_name, name, type, label, company_id 
FROM wk_crm_field 
WHERE label = 234  -- 合同变更的 label
ORDER BY company_id, sorting;

-- 如果为空，需要执行初始化脚本
-- 参考：DB/20260608/init_contract_change_fields.sql
```

**field_id 生成规则**：
- Java 代码使用 `BaseUtil.getNextId()`（雪花ID）
- SQL 脚本使用 `9000000000000000000 + (company_id % 10000) * 100 + 序号` 确保唯一

**常见错误**：
- `Field 'field_id' doesn't have a default value` → 需要手动指定 field_id
- 前端表单为空 → 检查 `wk_crm_field` 表中是否有 `label=模块类型 AND company_id=当前企业` 的记录

## 审批流字段权限的两套系统

| 系统 | 后端核心类 | 前端核心组件 | 说明 |
|------|-----------|------------|------|
| 旧式字段授权 | ExamineCrmServiceImpl | ExamineInfoSection + CRMAllCreate | 控制审批人能看到/编辑哪些业务字段 |
| 新式节点字段 | FlowFormFieldServiceImpl | FlowFormFieldDialog | 审批节点级别的自定义填写字段 |

### queryExamineField 的类型过滤

**文件**：`crm/crm-web/.../CrmFieldServiceImpl.java`

`queryExamineField()` 方法中定义了审批流字段权限配置可用的字段类型白名单：

```java
List<Integer> typeList = Arrays.asList(
    FieldEnum.SELECT.getType(),      // 3  单选
    FieldEnum.FLOATNUMBER.getType(), // 6  小数
    FieldEnum.SYS_DICT.getType()     // 115 数据字典
    // NUMBER(5)、CHECKBOX(9)、PERCENT(42)、DATE(4)、DATETIME(13)、
    // USER(10)、STRUCTURE(12) 等当前被注释掉，不展示
);
```

**结论**：
- 默认只有 **SELECT(3)、FLOATNUMBER(6)、SYS_DICT(115)** 类型的字段会出现在审批流字段权限配置中
- TEXT / NUMBER / DATE / USER 等类型当前被注释掉，不会在审批节点字段权限配置界面中展示
- 如果需要在审批流中暴露更多字段类型，需要修改 `queryExamineField()` 方法的 `typeList`
- 此限制不影响表单正常显示，只影响"审批节点字段权限配置"界面

## 验证方法

修复后验证步骤：
1. 进入审批流配置 → 选择合同变更 → 审批人节点 → 字段权限，确认能看到业务字段
2. 进入客户管理 → 自定义字段设置 → 合同变更管理，确认字段列表正常
3. 提交一个合同变更审批 → 审批人打开审批面板 → 点击"详情"，确认能看到业务数据
