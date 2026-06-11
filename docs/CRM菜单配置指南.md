# CRM 菜单配置指南

本文档说明悟空CRM中集定制版中，新模块菜单注册的完整流程与核心规则，是后续开发人员配置新菜单的权威参考指南。

---

## 一、核心原则：权限路径必须前后端一致

菜单配置的首要规则是：**后端权限树的路径必须与前端路由的 `permissions` 数组一致**。

### 1.1 权限路径的结构

权限路径由 `wk_admin_menu` 表中的 `realm` 字段逐层串联构成：

```
权限路径 = CRM根菜单.realm → 模块目录.realm → 操作按钮.realm
```

示例：`crm → jtContractChange → index` 对应三层菜单的 realm 值。

### 1.2 后端权限树构建

后端 `AdminRoleServiceImpl.createMenu()` 方法（文件：`admin/admin-web/src/main/java/com/kakarote/admin/service/impl/AdminRoleServiceImpl.java` 第1327行）根据 `wk_admin_menu` 表的 `parent_id` 递归构建嵌套权限树：

```java
private JSONObject createMenu(Set<AdminMenu> adminMenuList, Long parentId) {
    JSONObject jsonObject = new JSONObject();
    adminMenuList.forEach(adminMenu -> {
        if (Objects.equals(parentId, adminMenu.getParentId())) {
            if (Objects.equals(1, adminMenu.getMenuType())) {
                // 目录(menu_type=1)：创建嵌套对象，key=realm
                JSONObject object = createMenu(adminMenuList, adminMenu.getMenuId());
                if (!object.isEmpty()) {
                    jsonObject.put(adminMenu.getRealm(), object);
                }
            } else {
                // 页面/按钮(menu_type=2/3)：设为true，key=realm
                jsonObject.put(adminMenu.getRealm(), Boolean.TRUE);
            }
        }
    });
    return jsonObject;
}
```

构建出的权限树示例：

```
authInfo = {
  crm: {                           // menu_id=1, parent_id=0, realm='crm', menu_type=1
    leads: {                       // menu_id=9, parent_id=1, realm='leads', menu_type=1
      index: true                  // menu_id=19, parent_id=9, realm='index', menu_type=3
    },
    customer: {
      index: true
    },
    jtContract: {                  // menu_id=2400, parent_id=1, realm='jtContract', menu_type=1
      index: true                  // menu_id=2403, parent_id=2400, realm='index', menu_type=3
    },
    jtContractChange: {            // menu_id=4700, parent_id=1, realm='jtContractChange', menu_type=1
      index: true                  // menu_id=4703, parent_id=4700, realm='index', menu_type=3
    }
  }
}
```

### 1.3 前端权限检查

前端权限检查流程（文件：`dz-zhongji/src/store/utils/index.js`）：

1. 用户登录后，前端调用 `adminRole/auth` 获取权限数据 `authInfo`
2. `filterAsyncRouter(routers, authInfo)` 遍历每个路由的 `meta.permissions`
3. `forCheckPermission(['crm', 'jtContractChange', 'index'], authInfo)` 按路径逐层查找：

```javascript
const forCheckPermission = function(permissions, authInfo) {
  for (let index = 0; index < permissions.length; index++) {
    const key = permissions[index]
    authInfo = authInfo[key]
    if (!authInfo) {
      return false  // 任一层找不到 → 权限不通过
    } else if (permissions.length - 1 === index) {
      return true   // 所有层都找到 → 权限通过
    }
  }
}
```

4. 如果 `authInfo.crm.jtContractChange.index` 不存在，该路由被过滤掉
5. 被过滤的路由不会出现在导航菜单中，直接访问也会404

**核心结论**：权限路径的每一段（`crm`、`jtContractChange`、`index`）都必须在后端 `wk_admin_menu` 表中有对应 `realm` 的菜单记录，且层级关系完全一致。

---

## 二、常见错误案例分析

### 2.1 错误：将新模块挂在二级目录下

新模块菜单**不能**挂在合同管理(menu_id=13)等二级目录下，否则权限路径多出一级：

```
错误配置: parent_id=13 → 权限路径 crm.contract.contractChange.index
前端期望: permissions ['crm', 'jtContractChange', 'index'] → 权限路径 crm.jtContractChange.index
两者不一致，前端路由被权限过滤掉，菜单不显示！
```

正确配置：`parent_id=1`（CRM根菜单），权限路径直接是 `crm.jtContractChange.xxx`。

### 2.2 错误：CrmEnum 的 remarks 与菜单 realm 不一致

`CrmEnum` 的第二个参数 `remarks` 必须与菜单目录的 `realm` 完全一致。如果写成 `contractChange` 而菜单 realm 是 `jtContractChange`，会导致：
- ES索引名拼接错误（`wukong_contractChange` 而不是 `wukong_jtContractChange`）
- 表名前缀逻辑异常（`isZjBusinessTable` 判断错误）

### 2.3 错误：CrmAuthEnum 菜单基址与 wk_admin_menu ID不匹配

`CrmAuthEnum.getStandardAuthMenuId()` 中的基址必须与 `wk_admin_menu` 中目录的 `menu_id` 一致。例如，如果目录 `menu_id=4700` 而代码写 `4700L + value`，则 ADD=4701, EDIT=4702, LIST=4703 等必须与数据库中按钮的 `menu_id` 对应。任何偏差都会导致权限校验失败。

### 2.4 错误：wk_crm_field 字段 type 与 CrmFieldConst 定义不一致

当 `wk_crm_field` 表中字段 `type` 与 `CrmFieldConst.queryInitField()` 定义不一致时（如数据库 `type=1(TEXT)` 而代码定义 `DATA_UNION`），`queryListHead` API 会返回数据库的 type，导致前端 `formType` 不匹配，字段渲染异常。详见第六节"字段配置双表协同机制"。

### 2.5 错误：角色未授权新菜单

菜单和代码都配置正确，但当前用户的角色没有在 `wk_admin_role_menu` 中授权新菜单的 `menu_id`，导致权限树中没有对应节点，前端路由被过滤掉。这是最常见的遗漏。

---

## 三、菜单注册完整5步流程

新CRM模块上线需完成以下5步，缺一不可。

### 第1步：wk_admin_menu 菜单注册（数据库）

#### 表结构

`wk_admin_menu` 表的关键字段：

| 字段 | 类型 | 说明 | 规则 |
|------|------|------|------|
| `menu_id` | Long | 菜单ID | 按CrmAuthEnum标准模式连续分配：目录=N, save=N+1, update=N+2, index=N+3, read=N+4, delete=N+5, transfer=N+6 |
| `parent_id` | Long | 上级菜单ID | **目录必须为1**（CRM根菜单），不能挂在二级目录下 |
| `menu_name` | String | 菜单名称 | 中文显示名称，如"合同变更管理" |
| `realm` | String | 权限标识 | 目录的realm必须与前端路由permissions第二段一致（如 `jtContractChange`）；操作按钮realm固定：`save/update/index/read/delete/transfer` |
| `realm_url` | String | 权限URL | 操作按钮对应的后端接口路径 |
| `menu_type` | Integer | 菜单类型 | 目录=1, 页面路由=2, 操作按钮=3 |
| `sort` | Integer | 排序 | CRM根菜单下的排序位（避开已有模块的sort值） |
| `status` | Integer | 状态 | 1=启用, 0=禁用 |

#### ID分配规则

菜单ID按 `目录ID + CrmAuthEnum.value` 连续分配：

| CrmAuthEnum值 | 含义 | menu_id |
|---------------|------|---------|
| 目录 | 模块目录 | N |
| ADD=1 | 新建 | N+1 |
| EDIT=2 | 编辑 | N+2 |
| LIST=3 | 查看列表 | N+3 |
| READ=4 | 查看详情 | N+4 |
| DELETE=5 | 删除 | N+5 |
| TRANSFER=6 | 转移 | N+6 |

非标准按钮（如"作废"、"产品配置查询"等）ID从 `N+7` 开始分配。

#### SQL示例（合同变更 jtContractChange，基址4700）

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

-- 非标准按钮（ID从4707开始）
INSERT INTO wk_admin_menu (menu_id, parent_id, menu_name, realm, realm_url, menu_type, sort, status) VALUES
(4707, 4700, '作废', 'voidChange', '/crmJtContractChange/void', 3, 7, 1);
```

#### 角色授权（必须！）

菜单注册后，必须为当前用户的角色授权：

```sql
-- 为角色授权新菜单（假设角色ID为X）
INSERT INTO wk_admin_role_menu (role_id, menu_id) VALUES
(X, 4700), (X, 4701), (X, 4702), (X, 4703), (X, 4704), (X, 4705), (X, 4706);
```

> **注意**：admin超级管理员拥有所有权限，无需单独授权。但普通角色必须授权才能看到新菜单。

### 第2步：CrmAuthEnum 菜单ID映射（Java代码）

文件：`crm/crm-web/src/main/java/com/kakarote/crm/constant/CrmAuthEnum.java`

在 `getStandardAuthMenuId()` 方法中添加映射（约第358行），格式为 `目录ID + value`：

```java
case JT_CONTRACT_CHANGE -> 4700L + value; // 菜单基址4700
```

其中 `value` 对应 CrmAuthEnum 的操作枚举值：

| 枚举 | value | 对应menu_id |
|------|-------|-------------|
| ADD | 1 | 4701 |
| EDIT | 2 | 4702 |
| LIST | 3 | 4703 |
| READ | 4 | 4704 |
| DELETE | 5 | 4705 |
| TRANSFER | 6 | 4706 |

**已分配基址对照**：

| CrmEnum | 基址 |
|---------|------|
| LEADS | 16 |
| CUSTOMER | 25 |
| CONTACTS | 39 |
| PRODUCT | 64 |
| BUSINESS | 45 |
| CONTRACT | 52 |
| RECEIVABLES | 59 |
| JT_CONTRACT | 2400 |
| KH | 2500 |
| JT_ORDER | 2700 |
| PRICE_APPLY | 4100 |
| BIDDING | 4665 |
| JT_CONTRACT_CHANGE | 4700 |
| JT_CONTRACT_CANCEL | 5000 |

> 新模块基址应选择未占用的百位数区间，避免与已有模块冲突。

### 第3步：CrmEnum 业务实体枚举（Java代码）

文件：`crm/crm-common/src/main/java/com/kakarote/crm/constant/CrmEnum.java`

添加枚举值：

```java
JT_CONTRACT_CHANGE(234, "jtContractChange"),  // 合同变更
```

参数说明：

| 参数 | 含义 | 规则 |
|------|------|------|
| 第1个 `type` | 模块类型/label值 | 与 `wk_crm_field` 表的 `label` 字段一致，是全局唯一的模块标识数字 |
| 第2个 `remarks` | 模块备注/realm | **必须与菜单目录的 realm 一致**，也影响ES索引名和数据库表名 |
| 第3个 `isInitIndex` | 是否初始化ES索引 | 默认true（缺省时），false表示不自动创建ES索引 |

`remarks` 的三个关键用途：

1. **权限路径**：与 `wk_admin_menu.realm` 构成权限树路径
2. **ES索引名**：`CrmEnum.getTableName()` 拼接为 `wukong_{remarks}`（如 `wukong_jtContractChange`）
3. **数据库表名**：当 `isZjBusinessTable=true` 时加 `zj_` 前缀

还需在 `getMainFieldName()` 中添加该模块的编号字段名（约第270行）：

```java
case JT_CONTRACT_CHANGE -> "contractChangeNum";
```

### 第4步：wk_crm_field 字段注册（数据库）

#### 字段注册的双表协同机制

CRM模块的字段行为依赖 `wk_crm_field` 与 `wk_crm_field_sort` 两张表协同：

- **`wk_crm_field`**：按 `company_id` 存储各公司独立的字段定义（`company_id=0` 为系统默认，`company_id=实际公司ID` 为业务公司配置），`label` 值与 `CrmEnum.type` 一致
- **`wk_crm_field_sort`**：按 `label + user_id` 控制该模块下字段的列表排序与可见性（`is_hide`, `is_lock`, `sort`）

二者缺一不可：
- `wk_crm_field` 缺失 → ES索引失败、查询无字段、数据写入异常
- `wk_crm_field_sort` 缺失 → 列表表头不显示

#### queryFieldByCache 合并逻辑

`CrmFieldServiceImpl.queryFieldByCache()`（第1251行）合并 `CrmFieldConst` 和 `wk_crm_field` 数据：

```java
List<ModelField> modelFields = CrmFieldConst.queryInitField(crmEnum); // 代码定义
Map<String, ModelField> fieldMap = ...; // 按fieldName建索引
for (CrmField crmField : crmFields) {  // 数据库记录
    String fieldName = StrUtil.toCamelCase(crmField.getFieldName());
    if (fieldMap.containsKey(fieldName)) {
        continue; // CrmFieldConst中已有同名字段 → 优先使用代码定义，跳过数据库记录
    }
    // 数据库中独有的字段 → 补充到结果列表
    modelFields.add(modelField);
}
```

**重要结论**：`CrmFieldConst` 中定义的字段优先级高于 `wk_crm_field` 表。但如果 `wk_crm_field` 表中同字段的 `type` 值与代码定义不一致，`queryListHead` API 仍然使用数据库的 `type`（因为 `queryListHead` 走的是 `wk_crm_field_sort` + `wk_crm_field` 联查）。

#### queryListHead 的字段来源

`CrmFieldSortServiceImpl.queryListHead()` 流程：

1. 检查 `wk_crm_field_sort` 是否有该用户的排序记录
2. 没有则调用 `saveUserFieldSort()` → `queryAllFieldSortList()` 创建初始排序
3. `queryAllFieldSortList()` 合并 `wk_crm_field` 表数据 + `getDefaultField()` 代码定义
4. SQL 从 `wk_crm_field_sort` 联查 `wk_crm_field`，返回字段的 `type` 来自数据库
5. `recordToFormType()` 将 `type` 数字转换为 `formType` 字符串

**所以**：`wk_crm_field` 表中的 `type` 值直接决定前端列表的 `formType`，如果 `type` 与 `CrmFieldConst` 定义不一致，前端渲染会出现异常。

#### wk_crm_field 注册SQL示例

```sql
-- 合同变更字段注册（label=234，company_id需替换为实际公司ID）
INSERT INTO wk_crm_field (field_id, label, field_name, name, type, field_type, company_id, is_hidden, operating) VALUES
(FIELD_ID_1, 234, 'contract_change_num', '变更单编号', 30, 1, COMPANY_ID, 0, 1),  -- type=30对应SERIAL_NUMBER
(FIELD_ID_2, 234, 'change_type', '变更类型', 1, 1, COMPANY_ID, 0, 1),             -- type=1对应TEXT
(FIELD_ID_3, 234, 'contract_num', '合同编号', 1, 1, COMPANY_ID, 0, 1),
(FIELD_ID_4, 234, 'kh_name', '客户名称', 27, 1, COMPANY_ID, 0, 1),                -- type=27对应DATA_UNION
(FIELD_ID_5, 234, 'owner_user_id', '业务经理', 10, 1, COMPANY_ID, 0, 1),            -- type=10对应USER
(FIELD_ID_6, 234, 'car_count', '台数', 5, 1, COMPANY_ID, 0, 1),                   -- type=5对应NUMBER
(FIELD_ID_7, 234, 'expected_change_completion_at', '变更期望完成时间', 4, 1, COMPANY_ID, 0, 1), -- type=4对应DATE
(FIELD_ID_8, 234, 'config_price_diff_total', '配置差价合计', 6, 1, COMPANY_ID, 0, 1), -- type=6对应FLOATNUMBER
(FIELD_ID_9, 234, 'check_status', '审核状态', 5, 1, COMPANY_ID, 0, 1);

-- 同样需要为company_id=0（系统默认）注册一份，或在部署时做跨公司复制
```

> **FieldEnum type 数字对照**：TEXT=1, NUMBER=5, FLOATNUMBER=6, DATE=4, USER=3, SERIAL_NUMBER=30, DATA_UNION=27, SELECT=2, DETAIL_TABLE=31, FILE=8 等。

### 第5步：前端路由配置

#### 5a. 路由注册

文件：`dz-zhongji/src/router/modules/crm.js`

```javascript
{
  ...layout({
    title: '合同',                        // 导航标签显示的标题（硬编码中文，不用i18n）
    icon: 'contract-line',
    permissionList: [
      ['crm', 'jtContract', 'index'],
      ['crm', 'jtContractCancel', 'index'],
      ['crm', 'jtContractChange', 'index']   // ← 与菜单realm一致
    ]
  }),
  children: [{
    path: 'contract/subs/jtContractChange',
    component: () => import('@/views/crm/contractChange'),
    meta: {
      requiresAuth: true,
      permissions: ['crm', 'jtContractChange', 'index'],  // ← 与菜单realm一致
      title: '合同变更管理',                  // 硬编码中文，禁止使用i18n.global.t()
      icon: 'contract-line'
    }
  }]
}
```

**路由配置要点**：
- `permissionList` 是OR逻辑（任一权限匹配即显示该导航标签）
- 子路由的 `permissions` 是AND逻辑（必须全部路径段匹配才可访问）
- `title` 字段必须使用硬编码中文字符串，与同模块其他路由风格一致

#### 5b. 模型定义

文件：`dz-zhongji/src/views/crm/model/crmTypeModel.js`

```javascript
{
  type: 234,                    // ← 与CrmEnum的type一致
  name: '合同变更',
  key: 'contractChange',        // ← 前端key可以和realm不同，但权限路径必须用realm值
  labelKey: 'crmJtContractChange',  // ← 后端API路径前缀
  primaryKey: 'jtContractChangeId', // ← 主键字段名
  mainFieldKey: 'contractChangeNum' // ← 编号字段名（与CrmEnum.getMainFieldName()一致）
}
```

**注意**：`key` 和 `realm` 可以不同（如 key=`contractChange` 而 realm=`jtContractChange`），但权限路径始终使用 `realm` 值。

---

## 四、权限检查机制详解

### 4.1 前端权限检查流程

1. 用户登录 → 调用 `adminRole/auth` → 返回权限数据 `authInfo`
2. `filterAsyncRouter(routers, authInfo)` 递归过滤路由：
   - 对每个路由调用 `checkAuth(router, authInfo)`
   - `checkAuth` 检查 `meta.permissions` 或 `meta.permissionList`
   - `forCheckPermission` 按路径逐层查找 `authInfo`
3. 权限不通过的路由被移除，不会出现在导航菜单中
4. 直接访问被过滤的URL会404

### 4.2 后端权限树构建流程

1. `AdminRoleServiceImpl.auth()` 查询用户通过角色关联的所有菜单
2. 查询 `wk_admin_role_menu` 获取用户角色关联的 `menu_id` 列表
3. `createMenu(menuSet, parentId=0)` 递归构建嵌套 JSONObject
4. `menu_type=1`（目录）的菜单创建嵌套对象，key 为 `realm`
5. `menu_type=2/3`（页面/按钮）的菜单设为 `true`，key 为 `realm`

### 4.3 权限缓存机制

后端权限数据有Redis缓存：
- 缓存key：`USER_AUTH_CACHE_KET + userId`
- 修改菜单配置后，必须清除缓存才能生效
- 清除方式：重启服务、手动删除Redis key、或通过管理接口清除

---

## 五、字段配置双表协同机制

### 5.1 wk_crm_field 表

- 按 `company_id` 区分多租户字段定义
- `company_id=0`：系统默认字段模板
- `company_id=实际公司ID`：业务公司的字段配置
- `label` 字段与 `CrmEnum.type` 一致（如234）
- `type` 字段对应 `FieldEnum` 的数字编码，直接决定前端 `formType`
- `field_name` 存储下划线格式（如 `contract_change_num`），代码中转换为驼峰

### 5.2 wk_crm_field_sort 表

- 按 `label + user_id` 控制字段的列表显示排序
- 首次访问模块时通过 `saveUserFieldSort()` 自动创建
- 如果超管已配置过字段排序，新用户继承超管的配置
- `is_hide`：是否在列表中隐藏
- `is_lock`：是否锁定排序位置
- `sort`：显示排序顺序

### 5.3 CrmFieldConst 代码定义

- `CrmFieldConst.queryInitField(crmEnum)` 提供代码级的字段类型定义
- 在 `queryFieldByCache` 合并逻辑中，**代码定义优先级高于数据库同名字段**
- 但 `queryListHead` 联查走的是 `wk_crm_field_sort` + `wk_crm_field`，**数据库的 `type` 仍决定列表表头的 `formType`**

### 5.4 最佳实践：保持一致性

**强烈建议**：`wk_crm_field` 表中的字段 `type` 必须与 `CrmFieldConst` 代码定义一致。如果不一致：
- `queryFieldByCache` 返回代码定义的 `formType`（如 `data_union`）
- `queryListHead` 返回数据库的 `formType`（如 `text`）
- 导致数据写入和列表显示行为不一致

**修复不一致的方法**：
1. 更新 `wk_crm_field` 表中对应字段的 `type` 值
2. 同时更新 `wk_crm_field_sort` 表中对应记录（如果存在）
3. 清除Redis缓存（`CrmConst.ALL_FIELD_CACHE_NAME`）
4. 如前端有自定义渲染模板，确保兼容两种 formType

### 5.5 跨公司字段复制

`wk_crm_field` 需完成跨公司复制（从 `company_id=0` 到实际 `company_id`）才能保障功能完整。缺失特定公司的字段记录会导致：
- ES索引创建失败（无法确定字段列表）
- `queryPageList` 查询无字段配置
- 数据写入缺少字段校验

---

## 六、菜单配置检查清单

新模块上线前，逐项确认：

### 数据库层

- [ ] `wk_admin_menu` 中目录的 `parent_id=1`（CRM根菜单）
- [ ] 目录的 `realm` 与前端路由 `permissions[1]` 一致
- [ ] 操作按钮ID按 `目录ID + CrmAuthEnum.value` 连续分配
- [ ] 操作按钮的 `realm` 使用标准命名：`save/update/index/read/delete/transfer`
- [ ] 操作按钮的 `realm_url` 指向正确的后端接口
- [ ] `wk_admin_role_menu` 中当前用户的角色已授权新菜单的所有 `menu_id`
- [ ] `wk_crm_field` 中有字段注册，`label` 与 `CrmEnum.type` 一致
- [ ] `wk_crm_field` 中字段的 `type` 与 `CrmFieldConst` 代码定义一致
- [ ] `wk_crm_field` 中有 `company_id=0` 和实际公司ID两份记录

### Java代码层

- [ ] `CrmAuthEnum.getStandardAuthMenuId()` 中有对应的 case，基址与目录 `menu_id` 一致
- [ ] `CrmEnum` 中有对应的枚举值，第二个参数 `remarks` 与菜单 realm 一致
- [ ] `CrmEnum.getMainFieldName()` 中有编号字段名的映射
- [ ] `CrmFieldConst.queryInitField()` 中有该模块的字段定义
- [ ] `CrmHiddenFieldUtil` 中未意外隐藏该模块的字段

### 前端层

- [ ] 路由文件中有对应的路由定义
- [ ] 路由 `permissionList` 包含正确的权限路径
- [ ] 子路由的 `permissions` 与菜单 realm 一致
- [ ] 路由 `title` 使用硬编码中文（不用i18n）
- [ ] `crmTypeModel.js` 中有对应的模型定义
- [ ] `crmTypeModel.type` 与 `CrmEnum.type` 一致
- [ ] `crmTypeModel.labelKey` 与后端 API 路径前缀一致
- [ ] `crmTypeModel.primaryKey` 与后端实体主键字段名一致
- [ ] `crmTypeModel.mainFieldKey` 与 `CrmEnum.getMainFieldName()` 一致

### 缓存层

- [ ] Redis权限缓存已清除（`USER_AUTH_CACHE_KET + userId`）
- [ ] CRM字段缓存已清除（`CrmConst.ALL_FIELD_CACHE_NAME`）

---

## 七、菜单不显示排查步骤

按优先级依次排查：

### 7.1 检查浏览器权限数据

登录后在浏览器Console执行：

```javascript
JSON.parse(localStorage.getItem('authInfo'))
```

确认 `crm.jtContractChange.index` 存在且值为 `true`。

也可检查Vuex store：在Console执行 `store.state.auth` 或 `store.getters.allAuth`。

### 7.2 检查数据库菜单配置

```sql
-- 查看模块菜单树
SELECT menu_id, parent_id, menu_name, realm, menu_type, sort
FROM wk_admin_menu
WHERE realm = 'jtContractChange' OR menu_id IN (
    SELECT menu_id FROM wk_admin_menu WHERE parent_id IN (
        SELECT menu_id FROM wk_admin_menu WHERE realm = 'jtContractChange'
    )
)
ORDER BY parent_id, sort;

-- 关键检查项：
-- 1. 目录 parent_id=1
-- 2. 目录 realm='jtContractChange'
-- 3. 按钮 parent_id=目录menu_id
-- 4. 按钮realm=save/update/index/read/delete/transfer
```

### 7.3 检查角色授权

```sql
-- 查看当前用户角色是否授权新菜单
SELECT rm.role_id, rm.menu_id, m.realm, m.menu_name
FROM wk_admin_role_menu rm
JOIN wk_admin_menu m ON rm.menu_id = m.menu_id
WHERE rm.menu_id IN (4700, 4703)  -- 目录和查看列表
AND rm.role_id = 当前用户角色ID;
```

> admin超级管理员自动拥有所有权限，无需检查此项。

### 7.4 检查前端路由配置

- 打开浏览器开发者工具 → Network → 刷新页面
- 查看 `adminRole/auth` 接口返回的权限数据
- 检查前端路由的 `meta.permissions` 是否与权限树路径一致

### 7.5 检查缓存

后端权限数据有Redis缓存，修改菜单后需要清除：

```sql
-- 查看Redis缓存key（需要在Redis CLI中操作）
-- key格式：USER_AUTH_CACHE_KET + userId
-- 或直接重启服务使缓存失效
```

### 7.6 检查ES索引

如果列表页显示但数据为空：

```bash
# 查看ES索引是否存在
curl http://ES_HOST:9200/_cat/indices?v | grep jtContractChange

# 查看索引mapping
curl http://ES_HOST:9200/wukong_jt_contract_change/_mapping?pretty
```

---

## 八、已知模块菜单参数对照表

| 模块 | CrmEnum | type | realm | parent_id | 菜单基址 | 主键 | 编号字段 |
|------|---------|------|-------|-----------|----------|------|----------|
| 线索管理 | LEADS | 1 | leads | 1 | 16 | leadsId | — |
| 客户管理 | CUSTOMER | 2 | customer | 1 | 25 | customerId | — |
| 联系人管理 | CONTACTS | 3 | contacts | 1 | 39 | contactsId | — |
| 商机管理 | BUSINESS | 4 | business | 1 | 45 | businessId | — |
| 合同管理(标准) | CONTRACT | 5 | contract | 1 | 52 | contractId | — |
| 回款管理 | RECEIVABLES | 6 | receivables | 1 | 59 | receivablesId | — |
| 产品管理 | PRODUCT | 7 | product | 1 | 64 | productId | — |
| 回访 | RETURN_VISIT | 19 | returnVisit | 1 | 400 | — | — |
| 报价单 | QUOTATION | 26 | quotation | 1 | 152315 | quotationId | — |
| 外勤签到 | OUT_WORK_SIGN | 22 | outWorkSign | 1 | 213 | — | — |
| 发票 | INVOICE | 18 | invoice | 1 | 420 | invoiceId | — |
| 回款计划 | RECEIVABLES_PLAN | 8 | receivablesPlan | 1 | 936 | — | — |
| JT合同 | JT_CONTRACT | 205 | jtContract | 1 | 2400 | jtContractId | contractNum |
| 客户(KH) | KH | 206 | kh | 1 | 2500 | khId | — |
| JT订单 | JT_ORDER | 222 | jtOrder | 1 | 2700 | jtOrderId | orderNum |
| 蓄水订单 | WATER_TANKER_ORDER_APPLY | 221 | waterTankerOrderApply | 1 | 2800 | — | — |
| 价格申请 | PRICE_APPLY | 226 | priceApply | 1 | 4100 | priceApplyId | — |
| 商品库 | GOODS_REPOSITORY | 213 | goodsRepository | 1 | 3700 | — | — |
| 提车申请(国内) | CAR_PICKUP_APPLY | 210 | carPickupApply | 1 | 3400 | — | — |
| 航运订舱 | SHIPPING_BOOKING | 211 | shippingBooking | 1 | 3350 | — | — |
| 预开发票 | PRE_INVOICE_APPLY | 215 | preInvApply | 1 | 3000 | — | — |
| 预付款退款 | PREPAY_REFUND | 217 | prepayRefund | 1 | 2900 | — | — |
| 转库申请 | TRANSFER_REPOSITORY_APPLY | 214 | transferRepositoryApply | 1 | 3100 | — | — |
| 招投标 | BIDDING | 231 | bidding | 1 | 4665 | biddingId | biddingNum |
| 保证金 deposit | MARGIN_DEPOSIT_APPLY | 232 | marginDepositApply | 1 | 4685 | — | — |
| 正常开票 | NORMAL_INVOICE | 233 | normalInvoice | 1 | 4900 | — | normalInvoiceNum |
| 合同变更 | JT_CONTRACT_CHANGE | 234 | jtContractChange | 1 | 4700 | jtContractChangeId | contractChangeNum |
| LC申请 | LC_APPLY | 235 | lcApply | 1 | 3200 | — | — |
| LC单证申请 | LC_DOC_APPLY | 236 | lcDocApply | 1 | 3600 | — | — |
| LG申请 | LG_APPLY | 237 | lgApply | 1 | 2600 | — | — |
| AP申请 | AP_APPLY | 238 | apApply | 1 | 3300 | — | — |
| 报价申请 | QUOTE_APPLY | 239 | quoteApply | 1 | 3500 | — | — |
| 合同取消 | JT_CONTRACT_CANCEL | 240 | jtContractCancel | 1 | 5000 | — | — |
| 出口专项费用 | EXPORT_SPECIAL_EXPENSE | 241 | exportSpecialExpense | 1 | 5020 | — | — |
| 现车改制 | VEHICLE_MODIFICATION_APPLY | 242 | vehicleModificationApply | 1 | 5040 | — | vehicleModificationApplyNum |
| 交车握手及插单 | DELIVERY_HANDSHAKE_AND_ORDER_INSERTION | 227 | deliveryHandshakeAndOrderInsertion | 1 | 4000 | — | — |

> **基址分配规则**：标准模块使用小数字（16-64），中集定制模块使用大数字（2000+区间），新模块应选择未占用的百位区间。当前已用到5040，新模块建议从5100开始。

---

## 附录A：FieldEnum type 数字对照表

> ⚠️ **重要修正**：本附录历史版本存在多处 type 值错误（如 SELECT=2、USER=3 等）。
> **请以 `common/common-web/.../enums/FieldEnum.java` 源码为唯一权威来源。**
> 以下表格已根据源码修正：

| FieldEnum | type数字 | formType字符串 | 说明 |
|-----------|---------|---------------|------|
| TEXT | 1 | text | 单行文本 |
| TEXTAREA | 2 | textarea | 多行文本 |
| SELECT | 3 | select | 下拉选择 |
| DATE | 4 | date | 日期 |
| NUMBER | 5 | number | 数字 |
| FLOATNUMBER | 6 | floatnumber | 浮点数字 |
| MOBILE | 7 | mobile | 手机 |
| FILE | 8 | file | 文件 |
| CHECKBOX | 9 | checkbox | 多选 |
| USER | 10 | user | 人员选择 |
| ATTACHMENT | 11 | attachment | 附件 |
| STRUCTURE | 12 | structure | 部门选择 |
| DATETIME | 13 | datetime | 日期时间 |
| EMAIL | 14 | email | 邮件 |
| ADDRESS | 24 | address | 地址 |
| WEBSITE | 25 | website | 网址 |
| PIC | 29 | pic | 图片 |
| DESC_TEXT | 50 | desc_text | 描述文字 |
| SERIAL_NUMBER | 63 | serial_number | 自动编号 |
| FIELD_GROUP | 60 | field_group | 字段分组 |
| TAG | 61 | field_tag | 标签 |
| ATTENTION | 62 | field_attention | 关注度 |
| DETAIL_TABLE | 45 | detail_table | 明细表格 |
| AREA_POSITION | 43 | position | 地址（省市区） |
| CURRENT_POSITION | 44 | location | 定位 |
| DATE_INTERVAL | 48 | date_interval | 日期区间 |
| BOOLEAN_VALUE | 41 | boolean_value | 布尔值 |
| PERCENT | 42 | percent | 百分数 |
| HANDWRITING_SIGN | 46 | handwriting_sign | 手写签名 |
| OPTIONS_TYPE | 49 | options_type | 逻辑表单/选项字段 |
| RICH_TEXT_FORMAT | 70 | rich_text_format | 富文本 |
| DIVIDER | 83 | divider | 分割线 |
| MATRIX_SCALE | 81 | matrix_scale | 矩阵量表 |
| SORT | 82 | sort | 排序 |
| DATA_COLLAPSE | 76 | data_collapse | 折叠分隔线 |
| DATA_UNION | 100 | data_union | 数据关联 |
| VIDEO | 74 | video | 视频 |
| BAR_QR_CODE | 73 | bar_qr_code | 条码/二维码 |
| SYS_DICT | 115 | sysDict | 数据字典 |

---

## 附录B：关键代码文件索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|---------|
| 权限树构建 | `admin/admin-web/.../AdminRoleServiceImpl.java` | 1327 |
| 菜单ID映射 | `crm/crm-web/.../CrmAuthEnum.java` | 358 |
| 业务枚举定义 | `crm/crm-common/.../CrmEnum.java` | 40-94 |
| 字段代码定义 | `crm/crm-web/.../CrmFieldConst.java` | — |
| 字段合并逻辑 | `crm/crm-web/.../CrmFieldServiceImpl.java` | 1251 |
| 列表头查询 | `crm/crm-web/.../CrmFieldSortServiceImpl.java` | 67 |
| 字段排序初始化 | `crm/crm-web/.../CrmFieldSortServiceImpl.java` | 506 |
| 前端权限过滤 | `dz-zhongji/src/store/utils/index.js` | 41 |
| 前端路由配置 | `dz-zhongji/src/router/modules/crm.js` | — |
| 前端模型定义 | `dz-zhongji/src/views/crm/model/crmTypeModel.js` | — |
| 菜单表结构 | `admin/admin-web/.../AdminMenu.java` | — |
| 字段表结构 | `crm/crm-common/.../CrmField.java` | — |