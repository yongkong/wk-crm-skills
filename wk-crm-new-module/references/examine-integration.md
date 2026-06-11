# 审批集成模板

> L2+ 模块审批集成的标准代码模式。
> 数据来源：`CrmBiddingServiceImpl.java`、`CrmJtContractChangeServiceImpl.java`、`ExamineEnum.java`、`ExamineLabelUtils.java`、`ExamineInfoSection.vue` 源码。

---

## 1. ExamineEnum 注册

**文件**：`zhongjicheliang/examine/examine-web/.../constant/ExamineEnum.java`

```java
// 格式：ENUM_NAME(type, relType, "中文名称", ExamineModuleTypeEnum.Crm),
// type 和 relType 相同，均 = CrmEnum.type
{ENUM_NAME}({type}, {type}, "{模块中文名}", ExamineModuleTypeEnum.Crm),
```

**注意**：已分配的 ExamineEnum type 与 CrmEnum.type 一致（如 BIDDING=231, JT_CONTRACT_CHANGE=234）

---

## 2. ExamineLabelUtils 常量

**文件**：`zhongjicheliang/crm/crm-web/.../common/ExamineLabelUtils.java`

```java
public static final int {CONSTANT_NAME} = {type}; // {模块中文名}
```

> 新增后需在 `ExamineLabelUtils.java` 中添加常量，然后在 ServiceImpl 中引用：`ExamineLabelUtils.{CONSTANT_NAME}`

---

## 3. ServiceImpl 审批集成核心代码

### 3.1 依赖注入

```java
@Autowired
private ExamineService examineService;
```

### 3.2 addOrUpdate 中的审批提交流程

#### 编辑场景（entity.id != null）

```java
// 1. 获取审批流数据
ExamineRecordSaveBO examineRecordSaveBO = crmModel.getExamineFlowData();
ExamineRecordReturnVO examineData = null;

// 2. 编辑时的状态校验
// checkStatus == 1（审批通过）→ 不可编辑
// checkStatus == 8（作废）→ 不可编辑
// checkStatus ∈ {2, 4, 5, 10} 才允许编辑
if (oldObject.getCheckStatus() == 1) {
    throw new CrmException(CrmCodeEnum.CRM_CONTRACT_EXAMINE_PASS_HINT_ERROR, getLabel().getRemarks());
}
if (!Arrays.asList(2, 4, 5, 10).contains(oldObject.getCheckStatus())) {
    throw new CrmException(CrmCodeEnum.CRM_CONTRACT_EDIT_ERROR, getLabel().getRemarks());
}

// 3. 提交审批（checkStatus != 5 即草稿时）
if (entity.getCheckStatus() != null && entity.getCheckStatus() == 5) {
    entity.setCheckStatus(5); // 保存草稿，不提交审批
} else {
    this.supplementFieldInfo(ExamineLabelUtils.{CONSTANT}, dataId, oldObject.getExamineRecordId(), examineRecordSaveBO);
    examineRecordSaveBO.setTitle(entity.get{编号/名称字段}());
    examineData = examineService.addExamineRecord(examineRecordSaveBO).getData();
    entity.setExamineRecordId(examineData.getRecordId());
    entity.setCheckStatus(examineData.getExamineStatus());
}
```

#### 新建场景（entity.id == null）

```java
save(entity);

if (ObjectUtil.notEqual(entity.getCheckStatus(), 5)) {
    this.supplementFieldInfo(ExamineLabelUtils.{CONSTANT}, dataId, null, examineRecordSaveBO);
    examineRecordSaveBO.setTitle(entity.get{编号/名称字段}());
    // 组装 dataMap 供审批人查看字段
    List<FieldData> fieldData = fieldDataService.queryByDataId(getLabel(), dataId);
    Map<String, Object> bean = BeanUtil.beanToMap(entity);
    fieldData.forEach(data -> bean.put(data.getName(), data.getValue()));
    examineRecordSaveBO.setDataMap(bean);
    examineData = examineService.addExamineRecord(examineRecordSaveBO).getData();
    entity.setExamineRecordId(examineData.getRecordId());
    entity.setCheckStatus(examineData.getExamineStatus());
}

updateById(entity);
```

### 3.3 supplementFieldInfo 方法

接口 `MpAdvancedSearchService` 和 `CrmPageService` 已提供 default 实现：

```java
default void supplementFieldInfo(Integer label, Long typeId, Long recordId, ExamineRecordSaveBO examineRecordSaveBO) {
    examineRecordSaveBO.setLabel(label);       // 审批类型标签 = ExamineLabelUtils 常量
    examineRecordSaveBO.setTypeId(typeId);     // 业务主键ID
    examineRecordSaveBO.setRecordId(recordId); // 旧审批记录ID（编辑时传入，新建时传null）
    if (examineRecordSaveBO.getDataMap() != null) {
        examineRecordSaveBO.getDataMap().put("createUserId", UserUtil.getUserId());
    } else {
        examineRecordSaveBO.setDataMap(new JSONObject().fluentPut("createUserId", UserUtil.getUserId()));
    }
}
```

**通常不需要覆写**，直接调用 `this.supplementFieldInfo(...)` 即可。

### 3.4 checkStatus 状态码对照

| 值 | 含义 | 操作权限 |
|----|------|---------|
| 0 | 未提交 | 可编辑/可删除 |
| 1 | 审批通过 | 不可编辑 |
| 2 | 审批拒绝 | 可编辑（修改后重新提交） |
| 3 | 审批中 | 可查看/可撤回 |
| 4 | 已撤回 | 可编辑 |
| 5 | 草稿 | 可编辑/可删除 |
| 8 | 作废 | 不可编辑 |
| 10 | 归档 | 可查看 |

### 3.5 queryExamineField 类型过滤（重要！）

**文件**：`zhongjicheliang/crm/crm-web/.../CrmFieldServiceImpl.java`

审批流配置中"字段权限"只展示特定类型的字段：

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
- 如果需要在审批流中暴露更多字段类型（如 TEXT、NUMBER、USER），需要修改 `queryExamineField()` 方法的 `typeList`
- 此限制不影响表单正常显示，只影响"审批节点字段权限配置"界面

---

## 4. 前端审批注册（9 处，仅 L2+）

> 数据来源：`ExamineInfoSection.vue`、`AuthFieldsMixin.js`、`examineApproveParams.js`、`CRMAllCreate.vue` 源码。

### 4.1 ExamineInfoSection.vue（4 处）

**文件**：`src/components/Examine/ExamineInfoSection.vue`

| # | 位置 | 作用 | 添加内容 |
|---|------|------|---------|
| 1 | `isCRMExamine()` label 数组 | 启用 CRM 字段授权审批模式 | 在数组中添加 `{label}` |
| 2 | `examineClick()` → `createType` 映射 | 审批人点"详情"加载业务表单 | `{label}: '{moduleKey}'` |
| 3 | `examineClick()` → `crmLabel` 映射 | 审批详情中获取字段授权配置 | `{label}: {label}` |
| 4 | `getAuthFieldsParams()` → `mainLabel` 映射 | 构建获取授权字段列表参数 | `'{moduleKey}': {label}` |

> **isCRMExamine 当前完整 label 数组**（源码截取自 `ExamineInfoSection.vue` 第 792 行）：
> ```javascript
> return label >= 1 && label <= 3 || [22, 202, 204, 205, 206, 210, 211, 214, 215, 217, 222, 223, 224, 226, 227, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 300, 301].includes(label)
> ```
> 新模块必须将 label 加入此数组，否则审批面板使用简单按钮模式，不激活字段授权。

> **getAuthFieldsParams mainLabel 当前完整映射**（源码截取自 `ExamineInfoSection.vue`）：
> ```javascript
> contract: 6, receivables: 7, invoice: 18, quotation: 26,
> productTemplate: 202, announcementTemplate: 204, jtContract: 205, kh: 206,
> carPickupApply: 210, transferRepositoryApply: 214, jtOrder: 222,
> khCreditLineApply: 223, khNetworkAccessApply: 224, priceApply: 226,
> lcApply: 235, lcDocApply: 236, lgApply: 237, apApply: 238, quoteApply: 239,
> exportSpecialExpense: 241, vehicleModificationApply: 242, bidding: 231,
> marginDepositApply: 232, deliveryHandshakeAndOrderInsertion: 227,
> preInvApply: 215, prepayRefund: 217, normalInvoice: 233,
> jtContractCancel: 240, contractChange: 234, shippingBooking: 211
> ```

> **注意**：`crmLabel` 映射在 `examineClick()` 方法内部，与 `createType` 映射是**两个独立的映射对象**，不要遗漏。
> 对于中集定制模块（type>=200），`crmLabel` 的值通常等于 label 本身（如 `234: 234`）。

### 4.2 AuthFieldsMixin.js（1 处）

**文件**：`src/components/Examine/mixins/AuthFieldsMixin.js`

| # | 位置 | 作用 | 添加内容 |
|---|------|------|---------|
| 5 | `flowGetCRMDetailRequestFun()` createType→ReadAPI 映射 | 审批弹层请求业务数据 | `'{moduleKey}': readApi` |

### 4.3 examineApproveParams.js（3 处）

**文件**：`src/components/Examine/examineApproveParams.js`

| # | 映射常量 | 作用 | 添加内容 |
|---|---------|------|---------|
| 6 | `EXAMINE_INFO_APPROVE_LABEL_MAP` | 审批流 label→业务 label | `{审批label}: {业务label}` |
| 7 | `CRM_MESSAGE_APPROVE_LABEL_MAP` | 待办 model→业务 label | 未接入待办可跳过 |
| 8 | `CRM_MESSAGE_CREATE_TYPE_MAP` | 待办 model→createType key | 未接入待办可跳过 |

> **注意**：合同变更(234)和合同取消(240)当前未接入待办系统，所以 CRM_MESSAGE 映射中无对应条目。但其他模块如 bidding(231)、marginDepositApply(232) 等已接入，需按实际情况判断。

### 4.4 CRMAllCreate.vue（1 处，审批弹层组件注册）

**文件**：`src/views/crm/components/CRMAllCreate.vue`

| # | 位置 | 作用 | 添加内容 |
|---|------|------|---------|
| 9 | import + components + componentName 映射 | 审批弹层加载业务 Create 组件 | 导入组件 + 注册 + `'{moduleKey}': '{Module}Create'` |

```javascript
// 1. 导入
import {Module}Create from '@/views/crm/{module}/Create'
// 2. components 注册
components: { {Module}Create }
// 3. componentName 映射（computed 中）
{moduleKey}: '{Module}Create'
```

**遗漏后果**：审批人无法在审批面板中查看业务数据/字段权限配置，点击"详情"无法加载业务表单

---

## 5. 审批回调（Examine 模块侧）

审批通过后，Examine 模块通过 `ExamineModuleService` 回调 CRM 的 Service 方法更新 `checkStatus`。

**文件**：`zhongjicheliang/examine/examine-web/.../ExamineModuleService.java` 的 CRM 实现类

```java
// 回调中更新业务实体 checkStatus
entity.setCheckStatus(examineStatus);
updateById(entity);
```

---

## 6. Entity 必备审批字段

所有 L2+ 模块的 PO 实体类必须包含：

```java
@TableField("examine_record_id")
private Long examineRecordId;  // 审批记录ID

@TableField("check_status")
private Integer checkStatus;   // 审批状态（见状态码对照表）
```
