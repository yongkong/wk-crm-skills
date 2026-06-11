# 表单渲染模式参考

> CRM 模块前端表单的三种渲染模式。选择哪种模式直接影响 Create.vue 的代码结构。
> 数据来源：`dz-zhongji/src/views/crm/` 下各模块实现。

---

## 模式对比

| 模式 | 适用场景 | 代表模块 | 复杂度 |
|------|---------|---------|--------|
| A. 纯动态表单 | 所有字段标准渲染，无特殊交互 | 招投标(bidding)、保证金 | 低 |
| B. 动态表单 + 自定义区块 | 有独立数据体系的复杂区块（如子表、变更信息） | 合同变更(contractChange) | 高 |
| C. 动态表单 + 字段级 slot | 某些字段需要自定义渲染组件（如 VIN 选择器） | 合同变更(contractChange) | 中 |

**实际项目中**，合同变更同时使用了模式 B 和 C：申请内容区块用模式 C（字段级 slot 自定义 VIN、contractId），变更信息区块用模式 B（完全硬编码的表格区块）。

---

## 模式 A：纯动态表单

**特征**：
- 所有字段由后端 `filedGetFieldAPI` 返回的 `fieldList` 驱动
- slot 中仅处理**通用类型**（如关联字段），不针对具体字段名做特殊判断
- 无额外的硬编码 `el-form-item` 区块

**代码模板**：

```vue
<!-- Create.vue 核心结构 -->
<create-sections title="基本信息">
  <el-form ref="crmForm" :model="fieldForm" :rules="fieldRules" label-position="top">
    <wk-form-items
      v-for="(children, index) in fieldList"
      :key="index"
      :field-from="fieldForm"
      :field-list="children"
      @change="formChange"
      @table-change="(...args) => tableChange(...args, crmTypeModel.{module})">
      <template #default="{ data }">
        <!-- 仅处理通用关联类型，无字段级特殊判断 -->
        <crm-relative-cell
          v-if="data && crmRelativeFormTypes.includes(data.formType)"
          :model-value="fieldForm[data.field]"
          :disabled="data.disabled"
          :props="data.props"
          :data-type="data.formType"
          @value-change="otherChange($event, data)" />
      </template>
    </wk-form-items>
  </el-form>
</create-sections>
```

**适用条件**：
- 所有字段都是标准类型（text/select/date/number 等）
- 无需弹窗选择、自动计算等复杂交互
- 无独立数据源的子表/明细

---

## 模式 B：动态表单 + 自定义区块

**特征**：
- 基础字段由 `wk-form-items` 动态渲染
- 存在**独立的 `create-sections` 区块**，内部完全硬编码 `el-table`、`el-form-item`
- 这些区块有**独立数据源**（如 `changeDetailList`），不属于 `fieldList` 字段体系
- 区块头部有自定义操作按钮

**代码模板**：

```vue
<!-- 区块1：基础字段 - 动态渲染 -->
<create-sections title="申请内容">
  <el-form ref="crmForm" :model="fieldForm" :rules="fieldRules" label-position="top">
    <wk-form-items ...>
      <!-- 同模式 A -->
    </wk-form-items>
  </el-form>
</create-sections>

<!-- 区块2：自定义区块 - 完全硬编码 -->
<create-sections title="变更信息">
  <template #header>
    <el-button @click="showAddChangeItemDialog">添加变更项</el-button>
    <el-button @click="addProductConfigRow">添加产品配置</el-button>
  </template>
  
  <el-table :data="changeDetailList" border>
    <el-table-column label="变更明细项" prop="changeDesc" />
    <el-table-column label="合同信息">
      <template #default="{ row }">
        <!-- 自定义渲染 -->
      </template>
    </el-table-column>
    <el-table-column label="变更描述">
      <template #default="{ row }">
        <el-input v-if="row.inputType === 'text'" v-model="row.value" />
        <el-input-number v-else-if="row.inputType === 'number'" v-model="row.value" />
      </template>
    </el-table-column>
  </el-table>
</create-sections>
```

**适用条件**：
- 有独立数据源的子表/明细（如变更明细、产品配置）
- 需要复杂交互（弹窗选择、批量操作、动态增删行）
- 数据不存储在 `wk_crm_field` 体系中

**关键实现点**：
- 自定义区块的数据需要单独管理（如 `changeDetailList`）
- 保存时需要分别处理主表字段和自定义区块数据
- 编辑回显时需要从后端加载自定义区块数据

---

## 模式 C：动态表单 + 字段级 slot

**特征**：
- 字段仍属于 `fieldList` 体系（后端配置的自定义字段）
- 通过 `data.field` 精确匹配拦截，用自定义组件替代默认渲染
- slot 透传链路：`WkFormItems` → `WkFormItem` → `WkField`

**代码模板**：

```vue
<wk-form-items ...>
  <template #default="{ data }">
    <!-- 按 data.field 精确匹配，自定义渲染特定字段 -->
    
    <!-- 1. contractId 字段 → 自定义关联选择器 -->
    <crm-relative-cell
      v-if="data && data.field === 'contractId'"
      :model-value="fieldForm.contractId"
      :disabled="data.disabled || isEdit"
      :radio="true"
      data-type="jtContract"
      :props="contractRelativeProps"
      @value-change="onContractSlotChange($event, data)" />
    
    <!-- 2. vinList 字段 → 自定义 VIN 选择器（只读输入框 + 弹出按钮） -->
    <div v-else-if="data && data.field === 'vinList'" class="vin-input-with-button">
      <el-input
        :value="vinDisplayText"
        :disabled="data.disabled"
        readonly
        placeholder="请选择车辆VIN/流转单号" />
      <el-button
        v-if="!data.disabled"
        class="more-btn"
        icon="el-icon-plus"
        @click="openVehicleSelectDialog" />
    </div>
    
    <!-- 3. 计算字段 → 只读显示 -->
    <el-input
      v-else-if="data && data.field === 'configDiffAndFeeTotal'"
      :value="configDiffAndFeeTotalDisplay"
      disabled
      placeholder="自动计算" />
  </template>
</wk-form-items>
```

**适用条件**：
- 字段存储在 `wk_crm_field` 中，但需要自定义渲染
- 需要弹窗选择（如 VIN 选择、合同选择）
- 需要自动计算/联动（如选择合同后自动填充客户名称）
- 需要只读显示计算结果

**关键实现点**：
- slot 中用 `v-if/v-else-if` 按 `data.field` 精确匹配
- 自定义组件需要处理 `disabled`、`value-change` 等标准 props
- 联动逻辑需要在 `@value-change` 中实现

---

## 决策指南

```
是否需要自定义区块（独立数据源）？
├── 是 → 模式 B（动态表单 + 自定义区块）
└── 否 → 是否需要字段级自定义渲染？
    ├── 是 → 模式 C（动态表单 + 字段级 slot）
    └── 否 → 模式 A（纯动态表单）
```

**需求模板填写指引**：

| 场景 | 表单渲染模式 | 需要填写的信息 |
|------|-------------|---------------|
| 所有字段标准渲染 | A | 无需额外说明 |
| 有独立子表/明细区块 | B | 描述自定义区块的内容、字段、交互 |
| 某些字段需要弹窗/联动 | C | 列出需要自定义的字段名和渲染方式 |
| 既有子表又有字段自定义 | B + C | 两者都需要描述 |

---

## 参考实现

| 模块 | 文件路径 | 使用模式 |
|------|---------|---------|
| 招投标 | `views/crm/bidding/Create.vue` | A |
| 合同变更 | `views/crm/contractChange/Create.vue` | B + C |
| 保证金 | `views/crm/marginDeposit/Create.vue` | A |
| 出口专项费用 | `views/crm/exportSpecialExpense/Create.vue` | A |
