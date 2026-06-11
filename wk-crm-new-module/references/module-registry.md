# CRM 模块注册表

> 本文件是模块 type / 菜单基址的唯一权威来源。
> 新增模块后请同步更新此文件。
> 数据来源：`CRM菜单配置指南.md` §8 + 实际代码验证。

## 已分配 CrmEnum.type

| type | 枚举名 | realm | 模块中文名 |
|------|--------|-------|-----------|
| 1 | LEADS | leads | 线索管理 |
| 2 | CUSTOMER | customer | 客户管理 |
| 3 | CONTACTS | contacts | 联系人管理 |
| 4 | BUSINESS | business | 商机管理 |
| 5 | CONTRACT | contract | 合同管理(标准) |
| 6 | RECEIVABLES | receivables | 回款管理 |
| 7 | PRODUCT | product | 产品管理 |
| 8 | RECEIVABLES_PLAN | receivablesPlan | 回款计划 |
| 18 | INVOICE | invoice | 发票 |
| 19 | RETURN_VISIT | returnVisit | 回访 |
| 22 | OUT_WORK_SIGN | outWorkSign | 外勤签到 |
| 26 | QUOTATION | quotation | 报价单 |
| 205 | JT_CONTRACT | jtContract | JT合同 |
| 206 | KH | kh | 客户(KH) |
| 210 | CAR_PICKUP_APPLY | carPickupApply | 提车申请(国内) |
| 211 | SHIPPING_BOOKING | shippingBooking | 航运订舱 |
| 213 | GOODS_REPOSITORY | goodsRepository | 商品库 |
| 214 | TRANSFER_REPOSITORY_APPLY | transferRepositoryApply | 转库申请 |
| 215 | PRE_INVOICE_APPLY | preInvApply | 预开发票 |
| 217 | PREPAY_REFUND | prepayRefund | 预付款退款 |
| 221 | WATER_TANKER_ORDER_APPLY | waterTankerOrderApply | 蓄水订单 |
| 222 | JT_ORDER | jtOrder | JT订单 |
| 226 | PRICE_APPLY | priceApply | 价格申请 |
| 227 | DELIVERY_HANDSHAKE_AND_ORDER_INSERTION | deliveryHandshakeAndOrderInsertion | 交车握手及插单 |
| 232 | MARGIN_DEPOSIT_APPLY | marginDepositApply | 保证金 deposit |
| 231 | BIDDING | bidding | 招投标 |
| 233 | NORMAL_INVOICE | normalInvoice | 正常开票 |
| 234 | JT_CONTRACT_CHANGE | jtContractChange | 合同变更 |
| 235 | LC_APPLY | lcApply | 信用证条款申请 |
| 236 | LC_DOC_APPLY | lcDocApply | 信用证交单申请 |
| 237 | LG_APPLY | lgApply | 保函开立申请 |
| 238 | AP_APPLY | apApply | 账期申请 |
| 239 | QUOTE_APPLY | quoteApply | 报价单申请 |
| 240 | JT_CONTRACT_CANCEL | jtContractCancel | 合同取消 |
| 241 | EXPORT_SPECIAL_EXPENSE | exportSpecialExpense | 出口专项费用 |
| 242 | VEHICLE_MODIFICATION_APPLY | vehicleModificationApply | 现车改制 |

## 已分配菜单基址

| 基址 | 枚举名 | realm | 说明 |
|------|--------|-------|------|
| 16 | LEADS | leads | 标准模块 |
| 25 | CUSTOMER | customer | 标准模块 |
| 39 | CONTACTS | contacts | 标准模块 |
| 45 | BUSINESS | business | 标准模块 |
| 52 | CONTRACT | contract | 标准模块 |
| 59 | RECEIVABLES | receivables | 标准模块 |
| 64 | PRODUCT | product | 标准模块 |
| 936 | RECEIVABLES_PLAN | receivablesPlan | 标准模块 |
| 213 | OUT_WORK_SIGN | outWorkSign | 标准模块 |
| 400 | RETURN_VISIT | returnVisit | 标准模块 |
| 420 | INVOICE | invoice | 标准模块 |
| 152315 | QUOTATION | quotation | 标准模块 |
| 2400 | JT_CONTRACT | jtContract | 中集定制 |
| 2500 | KH | kh | 中集定制 |
| 2700 | JT_ORDER | jtOrder | 中集定制 |
| 2800 | WATER_TANKER_ORDER_APPLY | waterTankerOrderApply | 中集定制 |
| 2600 | LG_APPLY | lgApply | 中集定制 |
| 2900 | PREPAY_REFUND | prepayRefund | 中集定制 |
| 3000 | PRE_INVOICE_APPLY | preInvApply | 中集定制 |
| 3100 | TRANSFER_REPOSITORY_APPLY | transferRepositoryApply | 中集定制 |
| 3200 | LC_APPLY | lcApply | 中集定制 |
| 3300 | ANNOUNCEMENT_TEMPLATE | announcementTemplate | 中集定制 |
| 3350 | SHIPPING_BOOKING | shippingBooking | 中集定制 |
| 3400 | CAR_PICKUP_APPLY | carPickupApply | 中集定制 |
| 3500 | QUOTE_APPLY | quoteApply | 中集定制 |
| 3600 | LC_DOC_APPLY | lcDocApply | 中集定制 |
| 3700 | GOODS_REPOSITORY | goodsRepository | 中集定制 |
| 4000 | DELIVERY_HANDSHAKE_AND_ORDER_INSERTION | deliveryHandshakeAndOrderInsertion | 中集定制 |
| 4200 | DELIVERY_HANDSHAKE_AND_ORDER_INSERTION | deliveryHandshakeAndOrderInsertion | 中集定制 |
| 4300 | TRANSFER_REPOSITORY_APPLY | transferRepositoryApply | 中集定制 |
| 4400 | XC_RESOURCE_POOL | xcResourcePool | 中集定制 |
| 4500 | GZ_VEHICLE_POOL | gzVehiclePool | 中集定制 |
| 4600 | WAIT_SHIPMENT | waitShipment | 中集定制 |
| 4665 | BIDDING | bidding | 中集定制 |
| 4685 | MARGIN_DEPOSIT_APPLY | marginDepositApply | 中集定制 |
| 4700 | JT_CONTRACT_CHANGE | jtContractChange | 中集定制 |
| 4900 | NORMAL_INVOICE | normalInvoice | 中集定制 |
| 5000 | JT_CONTRACT_CANCEL | jtContractCancel | 中集定制 |
| 5020 | EXPORT_SPECIAL_EXPENSE | exportSpecialExpense | 中集定制 |
| 5040 | VEHICLE_MODIFICATION_APPLY | vehicleModificationApply | 中集定制 |

## 新模块分配建议

- **type 范围**：中集定制模块使用 200-299 区间，当前已用到 242，建议从 **250** 开始
- **基址范围**：当前已用到 5040，新模块建议从 **5100** 开始，每次递增 20（留足按钮空间）
- **realm 命名**：与 CrmEnum 枚举值的 camelCase 一致
- **parent_id**：所有新模块目录菜单必须 parent_id=1（CRM 根菜单）

## CrmAuthEnum 按钮标准偏移

| 偏移 | 操作 | realm 命名 | menu_type |
|------|------|-----------|-----------|
| +1 | 新建 | save | 3 |
| +2 | 编辑 | update | 3 |
| +3 | 查看列表 | index | 2（页面路由） |
| +4 | 查看详情 | read | 3 |
| +5 | 删除 | delete | 3 |
| +6 | 转移 | transfer | 3 |

> 非标准按钮（如"作废"）从 +7 开始分配，menu_type=3。
> 数据来源：`CRM菜单配置指南.md` §3 第2步 + `CrmAuthEnum.java`

## FieldEnum type 数字对照表

> **数据来源**：`common/common-web/.../enums/FieldEnum.java`（唯一权威来源）
> **注意**：`CRM菜单配置指南.md` 附录A 和旧版对照表中多处 type 值有误（如 SELECT=2、USER=3 等），以本表为准。本表数据直接来源于 `common/common-web/.../enums/FieldEnum.java` 源码。

| formType | type 数字 | 说明 |
|----------|----------|------|
| text | 1 | 单行文本 |
| textarea | 2 | 多行文本 |
| select | 3 | 单选（options存JSON数组） |
| date | 4 | 日期 |
| number | 5 | 数字 |
| floatnumber | 6 | 浮点数字 |
| file | 8 | 文件/附件 |
| checkbox | 9 | 多选 |
| user | 10 | 人员选择 |
| structure | 12 | 部门选择 |
| datetime | 13 | 日期时间 |
| address | 24 | 地址 |
| website | 25 | 网址 |
| pic | 29 | 图片 |
| mobile | 7 | 手机 |
| email | 14 | 邮箱 |
| desc_text | 50 | 描述文字 |
| serial_number | 63 | 自动编号（options存编号规则JSON） |
| field_group | 60 | 字段分组 |
| field_tag | 61 | 标签 |
| field_attention | 62 | 关注 |
| detail_table | 45 | 明细表格 |
| position | 43 | 地址（省市区） |
| location | 44 | 定位 |
| date_interval | 48 | 日期区间 |
| boolean_value | 41 | 布尔值 |
| percent | 42 | 百分数 |
| handwriting_sign | 46 | 手写签名 |
| options_type | 49 | 逻辑表单/选项字段 |
| rich_text_format | 70 | 富文本 |
| divider | 83 | 分割线 |
| matrix_scale | 81 | 矩阵量表 |
| sort | 82 | 排序 |
| data_collapse | 76 | 折叠分隔线 |
| data_union | 100 | 数据关联 |
| video | 74 | 视频 |
| bar_qr_code | 73 | 条码/二维码 |
| sysDict | 115 | 数据字典 |
| **中集定制关联类型** | | |
| shippingBooking | 211 | 海运订船 |
| preInvApply | 215 | 提前开票申请 |
| prepayRefund | 217 | 预收款退回 |
| kh | 260 | 中集客户 |
| jtContract | 261 | 车辆合同 |
| jtOrder | 222 | 车辆订单 |
| goodsRepository | 213 | 商品库 |
| normalInvoice | 233 | 正常开票 |
