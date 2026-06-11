# SQL 初始化模板

> 新模块数据库初始化的标准 SQL 模板。使用时将 `{变量}` 替换为参数卡片中的实际值。
> 数据来源：`CRM新增功能模块完整流程指南.md` §3 + `DB/20260608/init_contract_change_fields.sql`

---

## 1. wk_crm_field 初始化

### 通用模板

```sql
SET @uid = 0;
SET @base = {field_id_base};   -- 2042500000000000000 + (type * 1000)
SET @cid = {company_id};       -- 实际企业ID（如 2027277158278090752）
SET @label = {CrmEnum.type};

-- ========== SELECT 类型（type=3） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 3, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 50, NULL,
    '["选项1","选项2"]',
    @uid, @uid, NOW(), NOW());

-- ========== TEXT 类型（type=1） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 1, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', {maxLength}, NULL, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== DATE 类型（type=4） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 4, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 0, NULL, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== NUMBER 类型（type=5） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 5, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 10, NULL, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== FLOATNUMBER 类型（type=6） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 6, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 0, {精度}, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== TEXTAREA 类型（type=2） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 2, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 500, NULL, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== USER 类型（type=10） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 10, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 0, NULL, NULL,
    @uid, @uid, NOW(), NOW());

-- ========== CHECKBOX 类型（type=9） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 9, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 50, NULL,
    '["选项1","选项2"]',
    @uid, @uid, NOW(), NOW());

-- ========== DATA_UNION 类型（type=100） ==========
-- 中集定制关联字段，如关联合同(jtContract=261)、订单(jtOrder=222)等
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 100, 1, @label, @cid,
    0, {isRequired}, 3, {sortOrder}, '', 0, NULL,
    '{"type":{关联模块type}}',
    @uid, @uid, NOW(), NOW());

-- ========== SERIAL_NUMBER 类型（type=63） ==========
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
VALUES (@base + {seq}, '{fieldName}', '{显示名}', 63, 1, @label, @cid,
    0, 1, 3, {sortOrder}, '', 50, NULL,
    '{"prefix":"BG-","dateFormat":"yyMMdd","serialDigits":4,"resetCycle":"daily"}',
    @uid, @uid, NOW(), NOW());
```

### company_id=0 系统默认记录

每个字段需要**两份记录**：一份 company_id=0（系统默认），一份 company_id=实际企业ID。
生成实际企业 ID 记录后，用以下方式追加 company_id=0 副本：

```sql
INSERT INTO wk_crm_field (field_id, field_name, name, type, field_type, label, company_id,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, create_time, update_time)
SELECT field_id + 1000000000000000000, field_name, name, type, field_type, label, 0,
    is_hidden, is_null, operating, sorting, input_tips, max_length, precisions, options,
    create_user_id, update_user_id, NOW(), NOW()
FROM wk_crm_field WHERE label = @label AND company_id = @cid;
```

### 关键注意事项

- `field_id` 主键，BIGINT 有符号上限 `9223372036854775807`
- `field_id` 基础值 = `2042500000000000000 + (type * 1000)`，按模块隔离防冲突
- `company_id` 必须与实际企业ID匹配，否则前端查不到字段
- `create_user_id` 和 `update_user_id` 是必填字段，不能为 NULL
- SELECT 类型的 `options` 存 JSON 数组：`["选项1","选项2"]`
- SERIAL_NUMBER 类型的 `options` 存编号规则 JSON

---

## 2. wk_admin_menu 初始化

```sql
-- 目录菜单（menu_type=1，parent_id=1）
INSERT INTO wk_admin_menu (menu_id, parent_id, menu_name, realm, menu_type, sort, status)
VALUES ({基址}, 1, '{模块中文名}', '{realm}', 1, {排序}, 1);

-- 标准 CRUD 按钮（menu_type=3，ID = 基址 + CrmAuthEnum偏移）
INSERT INTO wk_admin_menu (menu_id, parent_id, menu_name, realm, realm_url, menu_type, sort, status)
VALUES
({基址+1}, {基址}, '新建', 'save', '/crm{Module}/addOrUpdate', 3, 1, 1),
({基址+2}, {基址}, '编辑', 'update', '/crm{Module}/addOrUpdate', 3, 2, 1),
({基址+3}, {基址}, '查看列表', 'index', '/crm{Module}/queryPageList', 2, 3, 1),
({基址+4}, {基址}, '查看详情', 'read', '/crm{Module}/queryById', 3, 4, 1),
({基址+5}, {基址}, '删除', 'delete', '/crm{Module}/deleteByIds', 3, 5, 1),
({基址+6}, {基址}, '转移', 'transfer', '/crm{Module}/changeOwnerUser', 3, 6, 1);
```

> 注意：`index` 按钮的 `menu_type=2`（页面路由），其他按钮均为 `menu_type=3`。
> 非标准按钮（如“作废”）从 `{基址+7}` 开始分配。

### 关键规则

- 列名必须使用 `menu_name`（非 `name`）、`realm_url`（非 `url`）、`realm_module`（非 `icon`）
- 目录 `parent_id=1`（CRM 根菜单），**绝不能挂在二级目录下**
- 按钮 `realm` 必须使用标准命名：save / update / index / read / delete / transfer
- 按钮 ID = 基址 + CrmAuthEnum 标准偏移值（+1~+6，非标准从 +7 开始）
- `index` 的 `menu_type=2`（页面路由），其他按钮 `menu_type=3`

---

## 3. wk_admin_role_menu 角色授权

```sql
-- 为管理员角色（role_id=183294931746816）授权所有新菜单（含 read 按钮共 7 条）
INSERT INTO wk_admin_role_menu (role_id, menu_id, company_id)
SELECT 183294931746816, menu_id, {company_id}
FROM wk_admin_menu WHERE menu_id BETWEEN {基址} AND {基址+6};
```

---

## 4. 回滚 SQL 模板

```sql
-- 回滚顺序：role_menu → menu → field_sort → field（逆序删除）
DELETE FROM wk_admin_role_menu WHERE menu_id BETWEEN {基址} AND {基址+10};
DELETE FROM wk_admin_menu WHERE menu_id BETWEEN {基址} AND {基址+10};
DELETE FROM wk_crm_field_sort WHERE label = {type};
DELETE FROM wk_crm_field WHERE label = {type};
```
