# 参考模块选择指南

> 根据模块特征选择最接近的已有模块作为参考实现。

---

## 按复杂度选择

| 复杂度 | 特征 | 推荐参考模块 |
|--------|------|-------------|
| **L1** | 无审批、无子表、纯 CRUD | `CrmStandardConfirmation` |
| **L2** | 有审批、无子表 | `CrmBiddingServiceImpl`（544行，标准模板） |
| **L3** | 有审批、有子表、有 ERP 同步 | `CrmJtContractChangeServiceImpl`（合同变更） |
| **L4** | 多子表、多 ERP、产品配置 | `CrmJtContractServiceImpl`（JT 合同） |

---

## 按业务特征选择

### 有审批流
- 简单审批（无子表）→ `CrmBiddingServiceImpl`
- 审批 + 子表 → `CrmJtContractChangeServiceImpl`
- 审批 + 状态机（驳回可编辑/作废/撤回）→ `CrmJtContractChangeServiceImpl`

### 有 ERP 集成
- 审批通过后同步 → `CrmJtContractChangeServiceImpl`
- 多工厂同步 → `CrmJtContractServiceImpl`

### 有关联选择
- 关联合同 → `CrmJtContractChangeServiceImpl`（jtContract 类型）
- 关联客户 → `CrmJtContractServiceImpl`（kh 类型）
- 关联订单 → `CrmJtOrderServiceImpl`（jtOrder 类型）

### 有子表
- 单子表 → `CrmJtContractChangeServiceImpl`
- 多子表 → `CrmJtContractServiceImpl`

### 有状态驱动操作按钮
- 标准审批状态 → `CrmBiddingServiceImpl`
- 复杂状态（审批状态 + 业务状态双轨）→ `CrmJtContractChangeServiceImpl`

---

## 已有模块清单

| 模块中文名 | Service 类名 | 复杂度 | 关键特征 |
|-----------|-------------|--------|---------|
| 招投标 | CrmBiddingServiceImpl | L2 | 标准审批，无子表 |
| 保证金 | CrmMarginDepositServiceImpl | L2 | 标准审批，无子表 |
| 合同变更 | CrmJtContractChangeServiceImpl | L3 | 审批+子表+ERP+状态驱动 |
| 合同取消 | CrmJtContractCancelServiceImpl | L2 | 审批+状态驱动 |
| 出口专项费用 | CrmExportSpecialExpenseServiceImpl | L2 | 标准审批 |
| 现车改制 | CrmVehicleModificationApplyServiceImpl | L3 | 审批+子表+ERP+双状态 |
| JT 合同 | CrmJtContractServiceImpl | L4 | 多子表+多ERP+产品配置 |
| 蓄水订单 | CrmWaterTankerOrderApplyServiceImpl | L3 | 审批+ERP |
| 商品库 | CrmGoodsRepositoryServiceImpl | L1 | 纯 CRUD |

---

## 选择建议

1. **优先选择同类型模块**：如果新模块与已有模块业务相似，优先选择该模块
2. **其次选择同复杂度模块**：如果无相似业务，选择同复杂度的标准模块
3. **L2 默认选招投标**：`CrmBiddingServiceImpl` 是 L2 标准模板，544 行，结构清晰
4. **L3 默认选合同变更**：`CrmJtContractChangeServiceImpl` 是 L3 标准模板
