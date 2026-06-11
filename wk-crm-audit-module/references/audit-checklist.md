# CRM 模块审计检查清单

> 完整的 5 类检查项清单，用于 `wk-crm-audit-module` 技能。
> 每项标注验证方法和权威源。

---

## 一、后端枚举注册（7 项）

### 1.1 CrmEnum 注册
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 模块枚举值存在 | grep `CrmEnum.java` 模块名 | `CrmEnum.java` |
| `getMainFieldName()` 有 case | grep 模块 type 值 | `CrmEnum.java` |
| `getTableName()` 返回正确表名 | 检查返回值 | `CrmEnum.java` |

### 1.2 ExamineEnum 注册（L2+ 必须）
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 审批枚举存在 | grep `ExamineEnum.java` 模块名 | `ExamineEnum.java` |
| type = relType = CrmEnum.type | 检查参数 | `ExamineEnum.java` |
| `ExamineModuleTypeEnum.Crm` | 检查枚举类型 | `ExamineEnum.java` |

### 1.3 CrmAuthEnum 注册
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 枚举存在 | grep `CrmAuthEnum.java` 模块名 | `CrmAuthEnum.java` |
| `getStandardAuthMenuId()` 有 case | grep type 值 | `CrmAuthEnum.java` |

### 1.4 CrmFieldConst 注册
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| `queryInitField()` 有 case | grep type 值 | `CrmFieldConst.java` |
| 初始化字段列表完整 | 对比需求模板 | `CrmFieldConst.java` |

### 1.5 CrmFieldSortServiceImpl 注册
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| `getDefaultField()` 有 case | grep type 值 | `CrmFieldSortServiceImpl.java` |
| 默认字段排序正确 | 检查字段列表 | `CrmFieldSortServiceImpl.java` |

### 1.6 CrmFieldServiceImpl 注册
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| `queryFields()` sortMap 有条目 | grep moduleKey | `CrmFieldServiceImpl.java` |
| 字段排序逻辑正确 | 检查 sortMap 内容 | `CrmFieldServiceImpl.java` |

### 1.7 ExamineLabelUtils 常量（L2+ 必须）
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 常量已定义 | grep `ExamineLabelUtils.java` 模块名 | `ExamineLabelUtils.java` |
| ServiceImpl 正确引用 | grep `ExamineLabelUtils.` | ServiceImpl.java |

### 1.8 CrmHiddenFieldUtil 检查
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 未被意外隐藏 | grep `CrmHiddenFieldUtil.java` 模块 type | `CrmHiddenFieldUtil.java` |
| 字段正常显示 | 前端列表/详情页字段是否完整 | 页面实际渲染 |

### 1.9 ES 索引检查（L3+）
| 检查项 | 验证方法 | 权威源 |
|--------|---------|--------|
| 索引存在 | `curl ES_HOST:9200/_cat/indices | grep {realm}` | Elasticsearch |
| mapping 正确 | `curl ES_HOST:9200/wukong_{realm}/_mapping` | CrmFieldConst 定义 |

---

## 二、数据库初始化数据（4 类）

### 2.1 wk_crm_field
| 检查项 | SQL 查询 | 预期 |
|--------|---------|------|
| 字段记录存在 | `SELECT COUNT(*) FROM wk_crm_field WHERE label = {type}` | > 0 |
| company_id=0 记录 | `SELECT COUNT(*) FROM wk_crm_field WHERE label = {type} AND company_id = 0` | > 0 |
| 实际公司 ID 记录 | `SELECT COUNT(*) FROM wk_crm_field WHERE label = {type} AND company_id > 0` | > 0 |
| field_id 无冲突 | `SELECT field_id, COUNT(*) FROM wk_crm_field GROUP BY field_id HAVING COUNT(*) > 1` | 无该模块的 field_id |
| type 值正确 | 对比 `references/module-registry.md` FieldEnum type | 一致 |
| SELECT options 合法 | `SELECT options FROM wk_crm_field WHERE label = {type} AND type = 3` | JSON 数组格式 |

### 2.2 wk_admin_menu
| 检查项 | SQL 查询 | 预期 |
|--------|---------|------|
| 目录存在 | `SELECT * FROM wk_admin_menu WHERE realm LIKE '%{module}%' AND menu_type = 1` | 1 条 |
| parent_id=1 | `SELECT parent_id FROM wk_admin_menu WHERE realm LIKE '%{module}%' AND menu_type = 1` | 1 |
| 标准 6 按钮 | `SELECT * FROM wk_admin_menu WHERE parent_id IN (SELECT menu_id FROM wk_admin_menu WHERE realm = '{module}' AND menu_type = 1)` | 6 条 |
| 按钮 realm 正确 | 检查 realm 格式 | `index`/`save`/`update`/`read`/`delete`/`transfer`（realm 本身不含模块前缀） |

### 2.3 wk_admin_role_menu
| 检查项 | SQL 查询 | 预期 |
|--------|---------|------|
| 管理员已授权 | `SELECT COUNT(*) FROM wk_admin_role_menu WHERE role_id = 1 AND menu_id IN (SELECT menu_id FROM wk_admin_menu WHERE realm LIKE '%{module}%')` | = 菜单数 |

### 2.4 wk_crm_field_sort
| 检查项 | SQL 查询 | 预期 |
|--------|---------|------|
| 排序记录存在 | `SELECT COUNT(*) FROM wk_crm_field_sort WHERE label = {type}` | > 0 |
| 排序与代码一致 | 对比 `CrmFieldSortServiceImpl` | 字段列表一致 |

---

## 三、后端 Service/Controller（4 类）

### 3.1 文件完整性
| 文件 | 路径模式 | 检查 |
|------|---------|------|
| PO | `entity/PO/zj/Crm{Module}.java` | 存在 |
| SaveBO | `entity/BO/zj/Crm{Module}SaveBO.java` | 存在 |
| VO | `entity/VO/zj/Crm{Module}VO.java` | 存在 |
| Mapper | `mapper/zj/Crm{Module}Mapper.java` | 存在 |
| Mapper XML | `mapper/zj/Crm{Module}Mapper.xml` | 存在 |
| Service | `service/zj/ICrm{Module}Service.java` | 存在 |
| ServiceImpl | `service/impl/zj/Crm{Module}ServiceImpl.java` | 存在 |
| Controller | `controller/zj/Crm{Module}Controller.java` | 存在 |

### 3.2 Service 方法（按复杂度）
| 复杂度 | 必须方法 |
|--------|---------|
| L1 | queryField, queryById, addOrUpdate, deleteByIds |
| L2 | L1 + changeOwnerUser, 审批提交, 审批回调 |
| L3 | L2 + 子表保存, ERP 同步 |
| L4 | L3 + 多子表, 产品配置 |

### 3.3 审批集成（L2+）
| 检查项 | 验证方法 |
|--------|---------|
| ExamineService 注入 | grep `@Autowired` + `ExamineService` |
| addOrUpdate 有审批逻辑 | grep `addExamineRecord` |
| supplementFieldInfo 调用 | grep `supplementFieldInfo` |
| 审批回调方法 | 检查 `ExamineModuleService` 实现 |

### 3.4 代码规范
| 检查项 | 预期 |
|--------|------|
| BO/PO/VO 分离 | 不混用 |
| 构造注入 | 非 @Autowired 字段注入 |
| @Schema 注解 | 完整 |
| 无魔法值 | 使用常量/枚举 |

---

## 四、前端代码（4 类）

### 4.1 注册层
| 检查项 | 文件 | 验证 |
|--------|------|------|
| crmTypeModel 定义 | `views/crm/model/crmTypeModel.js` | grep 模块名 |
| 路由配置 | `router/modules/crm.js` | grep 模块名 |
| API 文件 | `api/crm/{module}.js` | 13 个函数 |

### 4.2 页面层
| 检查项 | 文件 | 验证 |
|--------|------|------|
| 列表页 | `views/crm/{module}/index.vue` | 存在 |
| 新建/编辑页 | `views/crm/{module}/Create.vue` | 存在 |
| 详情页 | `views/crm/{module}/Detail.vue` | 存在（如需要） |
| 表单渲染模式 | Create.vue | A/B/C 正确 |

### 4.3 自定义字段注册（5 处）
| 位置 | 文件 | 检查 |
|------|------|------|
| 1 | `CrmFieldServiceImpl.java` | sortMap 条目（Phase 1 已查） |
| 2 | `views/admin/crm/customField/index.vue` | label→moduleType 映射 |
| 3 | `views/admin/crm/customField/index.vue` | 图标映射 |
| 4 | `views/admin/fields/index.vue` | title 映射 |
| 5 | `systemFields.js` | 系统字段中文名（可选） |

### 4.4 审批流前端注册（L2+，9 处）
| # | 文件 | 位置 |
|---|------|------|
| 1 | `ExamineInfoSection.vue` | isCRMExamine() label 数组 |
| 2 | `ExamineInfoSection.vue` | examineClick() createType 映射 |
| 3 | `ExamineInfoSection.vue` | examineClick() crmLabel 映射 |
| 4 | `ExamineInfoSection.vue` | getAuthFieldsParams() mainLabel 映射 |
| 5 | `AuthFieldsMixin.js` | flowGetCRMDetailRequestFun() ReadAPI |
| 6 | `examineApproveParams.js` | EXAMINE_INFO_APPROVE_LABEL_MAP |
| 7 | `examineApproveParams.js` | CRM_MESSAGE_APPROVE_LABEL_MAP |
| 8 | `examineApproveParams.js` | CRM_MESSAGE_CREATE_TYPE_MAP |
| 9 | `CRMAllCreate.vue` | 组件导入+注册+crmTypeMap |

---

## 五、审计报告模板

### 检查汇总

| 类别 | 检查项数 | ✓ 通过 | ✗ 失败 | ⚠ 警告 |
|------|---------|--------|--------|--------|
| 后端枚举注册 | 7 | | | |
| 数据库初始化 | 4类 | | | |
| 后端代码 | 4类 | | | |
| 前端代码 | 4类 | | | |
| **总计** | | | | |

### 问题分级

| 级别 | 定义 | 示例 |
|------|------|------|
| P0 阻塞 | 模块无法运行 | CrmEnum 未注册、表不存在 |
| P1 严重 | 核心功能缺失 | 审批流未集成、菜单不显示 |
| P2 一般 | 功能不完整 | 缺少按钮权限、字段排序异常 |
| P3 建议 | 代码规范问题 | 未使用构造注入、缺少注释 |

### 修复方案模板

```
问题：{描述}
级别：P{0-3}
影响：{功能异常描述}
修复：
  文件：{文件路径}
  修改：{具体代码/SQL}
回滚：{回滚方案}
```
