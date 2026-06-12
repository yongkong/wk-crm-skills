# Elasticsearch 使用全景分析

> 本文档基于项目源码和配置文件的深度分析，梳理 ES 在悟空 CRM 中集车辆定制版中的集成方式、使用场景、核心操作和最佳实践。

---

## 1. ES 集成架构

### 1.1 技术选型

| 项目 | 详情 |
|------|------|
| 客户端库 | `co.elastic.clients:elasticsearch-java`（官方 Java API Client） |
| ES 服务端版本 | **7.14.1**（Docker 镜像，含 analysis-icu + ik 插件） |
| 底层传输 | `RestClientTransport`（Apache HttpClient） |
| JSON 序列化 | `JacksonJsonpMapper` |
| 依赖位置 | `common/common-field/pom.xml` |

项目**未使用** Spring Data Elasticsearch 或 `RestHighLevelClient`，而是直接使用 Elastic 官方的新版 Java API Client。

### 1.2 配置加载链

```
Nacos (wk-es.yml)
    ↓ 被以下模块引用
    ├── crm-web/application.yml
    ├── email-web/application.yml
    └── km-web/application.yml

wk-es.yml 内容：
spring:
  elasticsearch:
    uris: 127.0.0.1:9200
    username: elastic
    password: MLLaetyL4cVgGgR1FUbD
```

配置属性类：`common/common-field/.../config/ElasticClientProperties.java`
- 支持：`uris`、`username`、`password`、`connectionTimeout`(15s)、`socketTimeout`(120s)、SSL（指纹/证书）

客户端 Bean：`common/common-field/.../config/FieldAutoConfiguration.java`
- `@Bean ElasticsearchClient`：支持基本认证 + SSL 两种安全模式
- 自动配置类被所有使用 `common-field` 的模块共享

---

## 2. 使用 ES 的模块清单

### 2.1 核心模块

| 模块 | 用途 | 核心类 |
|------|------|--------|
| **CRM** | 主力模块，所有业务实体的列表查询/搜索/筛选 | `crm-web/.../common/InitEsIndexRunner.java` |
| **Email** | 邮件/短信联系人数据索引 | `email-web/.../common/InitEsIndexRunner.java` |
| **KM** | 知识库页面搜索 | `km-web/.../common/InitEsIndexRunner.java` |
| **Examine** | 审批流字段配置 | `examine-web/.../FlowFieldConfigServiceImpl.java` |
| **BI** | 商业智能（通过 Feign 调用 CRM 的 ES 查询） | `BiService`（Feign 接口） |

### 2.2 CRM 模块 ES 索引列表（由 CrmEnum 控制）

索引命名规则：`wukong_{enum_name_lowercase}`（特殊：`wukong_customer_2023`）

**initIndex=true（服务启动时自动创建）**：

| 索引名 | 模块 | 说明 |
|--------|------|------|
| `wukong_activity` | 跟进记录 | 标准 CRM |
| `wukong_marketinginfo` | 营销信息 | 标准 CRM |
| `wukong_jtcontract` | 集团合同 | 中集定制 |
| `wukong_kh` | 客户(KH) | 中集定制 |
| `wukong_dealerlogoff` | 经销商退网 | 中集定制 |
| `wukong_carpickupapply` | 国内提车申请 | 中集定制 |
| `wukong_shippingbooking` | 海运订船管理 | 中集定制 |
| `wukong_overseascarpickupapply` | 国外提车申请 | 中集定制 |
| `wukong_goodsrepository` | 商品库 | 中集定制 |
| `wukong_transferrepositoryapply` | 转库申请 | 中集定制 |
| `wukong_nonstandardcontract` | 非标合同 | 中集定制 |
| `wukong_watertankerorderapply` | 蓄水车订单 | 中集定制 |
| `wukong_jtcontractchange` | 合同变更 | 中集定制 |
| ... | 更多模块见 `CrmEnum.java` | |

**initIndex=false（不自动创建，需已有索引）**：

leads、customer、contacts、product、business、contract、receivables、invoice 等标准 CRM 模块。

> **关键规则**：`CrmEnum.initIndex` 控制是否在服务启动时由 `InitEsIndexRunner` 自动创建索引。若为 `false`，索引不存在时前端列表查询会抛出 `index_not_found_exception`（503 错误）。

---

## 3. ES 的核心角色

ES 在项目中扮演**列表搜索引擎**角色（非数据分析平台）：

```
MySQL（主存储）
    ↓ 双写
ES（搜索加速层）← 列表页查询全部走 ES
    ↑
前端请求 → createQueryBuilder() → ES search → parseMap() → 响应
```

数据流向：
1. **写入**：业务数据先写 MySQL，再同步写入 ES（同步或异步）
2. **读取**：列表查询走 ES，详情页走 MySQL
3. **删除**：MySQL 删除后同步调用 `EsUtil.deleteData()`

---

## 4. 核心工具类：EsUtil

文件路径：`common/common-field/src/main/java/com/kakarote/common/field/utils/EsUtil.java`（2051 行）

### 4.1 索引管理操作

| 方法 | 说明 |
|------|------|
| `createIndex(mappingMap, settingsAnalysis, fieldLabel)` | 创建索引，设置 mapping + settings |
| `indexExist(fieldLabel)` | 判断索引是否存在 |
| `putMapping(mappingMap, fieldLabel)` | 更新索引 mapping |
| `getMapping(fieldLabel)` | 获取索引 mapping |
| `addField(fieldName, property, fieldLabel)` | 动态给索引添加字段 |
| `refresh(fieldLabel)` | 手动刷新索引 |

**索引默认配置**（`createIndex` 方法）：
- 分片数：3，副本数：1
- 字段上限：2000（标准模块）/ 10000（ModuleFieldLabel 类模块）
- 嵌套字段上限：100
- 分析器：`lowercase_normalizer`（keyword 小写归一化）

### 4.2 文档增删改操作

| 方法 | 说明 |
|------|------|
| `saveData(dataMap, fieldLabel)` | 单条写入（docAsUpsert） |
| `saveData(valueList, fieldLabel, isAdd)` | 批量写入（create 或 update） |
| `updateField(dataMap, id, fieldLabel)` | 单条多字段更新 |
| `updateField(dataMap, ids, fieldLabel)` | 批量多字段更新 |
| `deleteData(ids, fieldLabel)` | 按 ID 删除（单条或批量） |
| `deleteByQuery(request)` | 按条件删除 |
| `bulk(operationList)` | 通用批量提交（带1次重试） |

### 4.3 查询操作

| 方法 | 说明 |
|------|------|
| `search(SearchRequest)` | 通用搜索接口 |
| `count(query, fieldLabel)` | 按条件计数 |
| `getById(id, index)` | 按 ID 查询文档 |

### 4.4 查询构建器（QueryBuilder 封装）

| 查询类型 | 方法 | 用途 |
|---------|------|------|
| 精确匹配 | `termQuery()` / `termsQuery()` | 状态筛选、权限过滤 |
| 通配符 | `wildcardQuery()` / `prefixQuery()` | 全文搜索（`*keyword*`） |
| 范围查询 | `rangeQuery()` | 时间/数字范围筛选 |
| 全文匹配 | `matchQuery()` | 多行文本搜索 |
| 嵌套查询 | `nestedQuery()` / `nestedNotQuery()` | 产品明细等嵌套数据搜索 |
| 脚本查询 | `scriptQuery()` | Painless 脚本条件 |
| 存在性 | `existsQuery()` | 字段是否为空判断 |
| 正则 | `regexpQuery()` | 正则表达式匹配 |
| ID 查询 | `idsQuery()` | 按 ID 批量查 |

### 4.5 字段类型映射（FieldEnum → EsFieldTypeEnum → ES Property）

| EsFieldTypeEnum | ES 类型 | 说明 |
|----------------|---------|------|
| `KEYWORD` | `wildcard` + `.sort`(icu_collation_keyword) | 默认类型，支持中文排序 |
| `TEXT` | `text`(ik_max_word) + `.sort`(icu_collation_keyword) | 多行文本，中文分词 |
| `DATE` | `date`（yyyy-MM-dd \|\| yyyy-MM-dd HH:mm:ss） | 日期/日期时间 |
| `NUMBER` | `scaled_float`(scalingFactor=100) | 数字类型 |
| `NESTED` | `nested`(dynamic=false) | 嵌套对象（产品明细等） |
| `FLATTENED` | `flattened` | 字典/数据联合类型 |

`fieldEnum2EsType()` 映射规则：
- `TEXTAREA/RTF` → `TEXT`
- `DATE/DATETIME` → `DATE`
- `NUMBER/FLOATNUMBER/BOOLEAN_VALUE/PERCENT` → `NUMBER`
- `DETAIL_TABLE/CONTRACT/RECEIVABLES/INVOICE/JT_CONTRACT/GOODS_REPOSITORY` → `NESTED`
- `DATA_UNION/DATA_DICTIONARY/SYS_DICT` → `FLATTENED`
- 其他 → `KEYWORD`

---

## 5. 列表搜索流程（CrmPageService）

文件路径：`crm-web/src/main/java/com/kakarote/crm/service/CrmPageService.java`（1342 行）

`CrmPageService` 接口是所有 CRM 列表页的搜索规范，实现了 `FieldPageService<CrmSearchBO>`。

### 5.1 查询构建流程

```
前端请求 CrmSearchBO
    │
    ▼
preHandler()         ← 排序字段处理，>100页自动切换 search_after
    │
    ▼
createQueryBuilder() ← 核心：构建 BoolQuery
    ├── 全文搜索（appendSearch → wildcardQuery *keyword*）
    ├── 场景过滤（sceneQuery: 我的/下属的/关注的...）
    ├── 高级筛选（handlerSearchList → textSearch/numberSearch/dateSearch...）
    ├── 数据权限（setDataAuth → ownerUserId/teamMemberIds/examineUserList）
    └── 特殊条件（isTransform/poolId/commentType/isWxUser...）
    │
    ▼
EsUtil.search()      ← 执行 ES 查询
    │
    ▼
handlerResult()      ← 权限控制 + 字段掩码
    │
    ▼
parseMap()           ← 后处理（格式化金额/小数/审批信息等）
    │
    ▼
返回前端
```

### 5.2 数据权限过滤（setDataAuth）

```java
// 非管理员：按权限用户列表过滤
authBoolQuery.should(termsQuery("ownerUserId", dataAuthUserIds));
// 团队成员可见
authBoolQuery.should(termQuery("teamMemberIds", userId));
// 审批人可见
authBoolQuery.should(termQuery("examineUserList", userId));
// 公司隔离
boolQueryBuilder.filter(termQuery("companyId", UserUtil.getCompanyId()));
```

### 5.3 search_after 深度分页

当页码 > 100 页时，自动切换为 `search_after` 分页（通过 BI 服务），避免深分页性能问题。

---

## 6. 索引初始化流程（InitEsIndexRunner）

文件路径：`crm-web/src/main/java/com/kakarote/crm/common/InitEsIndexRunner.java`（1156 行）

### 6.1 启动时初始化

```java
@Override
public void run(ApplicationArguments args) {
    // 1. 遍历 CrmEnum，对 initIndex=true 且索引不存在的模块创建索引
    for (CrmEnum value : CrmEnum.values()) {
        if (value.isInitIndex() && !EsUtil.indexExist(value)) {
            initData(value, null);  // 创建索引 + 全量同步数据
        }
        // 2. 加载已有索引的 mapping 到静态缓存 mappingMap
        if (value.isInitIndex()) {
            GetMappingResponse mapping = EsUtil.getClient().indices().getMapping(...);
            mappingMap.put(k, map);  // 供后续搜索使用
        }
    }
    // 3. 初始化关注度和点赞信息
    saveUserStar(null);
    saveUserFavour(null);
    // 4. 初始化工商信息索引
    enterpriseService.initIndex();
}
```

### 6.2 全量数据同步

`initData()` 方法：
1. 创建索引 mapping（从 `wk_crm_field` + `CrmFieldConst.queryInitField()` 获取字段定义）
2. 用**50线程线程池**从 MySQL 分页查询数据，批量写入 ES
3. 额外处理：
   - 公海数据（`savePool`）
   - 团队成员（`saveTeamMembers`）
   - 阶段流程（`saveFlowData`）
   - 产品明细（`saveProductData`）
   - 通话记录/商机数量（仅客户模块）
   - 跟进记录关联数据（仅活动模块）

### 6.3 mappingMap 静态缓存

```java
public static Map<String, Map<String, Property>> mappingMap = new ConcurrentHashMap<>();
```

- 服务启动时从 ES 加载所有索引的 mapping
- 搜索时用于判断字段类型（text → `.keyword` 后缀，wildcard → `.sort` 后缀）
- **注意**：索引结构变更后需重启服务才能刷新此缓存

---

## 7. 冗余字段级联更新（ElasticUtil）

文件路径：`crm-web/src/main/java/com/kakarote/crm/common/ElasticUtil.java`（228 行）

当基础数据变更时，异步级联更新所有关联索引中的冗余名称字段：

| 触发变更 | 更新字段 | 影响索引 |
|---------|---------|---------|
| 用户改名 | `ownerUserName` / `createUserName` | leads/customer/contacts/contract/business/receivables/returnVisit/product/invoice |
| 部门改名 | `ownerDeptName` | leads/customer/contacts/contract/business/receivables/product/invoice |
| 客户改名 | `customerName` | contacts/business/contract/receivables/receivablesPlan/returnVisit/invoice/visitPlan/quotation |
| 联系人改名 | `contactsName` | contract/returnVisit/visitPlan/quotation |
| 商机改名 | `businessName` | contract/visitPlan/quotation |
| 合同改名 | `contractNum` | receivables/returnVisit/invoice |

**实现方式**：`ThreadPoolTaskExecutor` 异步执行，通过 MySQL 查询受影响的 ID 列表，再批量更新 ES。

---

## 8. 数据保存模式（savePage）

`CrmPageService.savePage()` 是业务数据写入 ES 的统一入口：

```java
default void savePage(CrmModelSaveBO model, Object id, boolean isExcel, FieldLabel label) {
    Map<String, Object> map = model.getEntity();
    // 1. 负责人变更时补充所属部门
    if (map.containsKey("ownerUserId")) {
        SimpleUser simpleUser = UserCacheUtil.getSimpleUser(ownerUserId);
        map.put("ownerDeptId", simpleUser.getDeptId());
        map.put("ownerDeptName", simpleUser.getDeptName());
    }
    // 2. 审批流数据
    // 3. 产品明细（商机保存时）
    // 4. 调用 FieldPageService.super.savePage() → EsUtil.saveData()
}
```

---

## 9. 异常处理与重试机制

| 场景 | 处理方式 |
|------|---------|
| bulk 批量失败 | 重试1次，仅重试失败的文档 ID |
| 单条 update 冲突 | `retryOnConflict(3)`（乐观锁冲突） |
| 404 文档不存在 | 静默忽略（更新/删除时） |
| 其他异常 | 统一抛出 `CrmException(SystemCodeEnum.SYSTEM_SERVER_ERROR)` |
| 索引不存在 | 搜索时抛出 `index_not_found_exception`（503） |

---

## 10. Docker 部署配置

### 10.1 镜像与插件

```
Dockerfile (docker/es/Dockerfile):
FROM elasticsearch:7.14.1
COPY plugins/analysis-icu → /usr/share/elasticsearch/plugins/analysis-icu
COPY plugins/ik           → /usr/share/elasticsearch/plugins/ik
```

- `analysis-icu`：提供 `icu_collation_keyword` 类型，支持中文拼音排序
- `analysis-ik`：中文分词器（`ik_max_word` 最细粒度分词）

### 10.2 docker-compose 关键配置（优化后）

```yaml
environment:
  - cluster.name=wukong-cluster       # 固定集群名
  - node.name=wukong-node-1           # 固定节点名
  - discovery.type=single-node        # 单节点模式
  - ES_JAVA_OPTS=-Xms1g -Xmx1g       # JVM 堆内存（建议生产≥2g）
  - bootstrap.memory_lock=true        # 锁定内存防 swap
ulimits:
  memlock: { soft: -1, hard: -1 }    # 配合 memory_lock
  nofile:  { soft: 65536, hard: 65536 } # 文件描述符上限
healthcheck:
  test: curl -sf ... /_cluster/health?wait_for_status=yellow
  start_period: 60s                  # 冷启动宽容期
```

---

## 11. 注意事项与最佳实践

### 11.1 开发注意事项

1. **新增字段后需重启服务**：`InitEsIndexRunner.mappingMap` 为静态缓存，新增 ES 字段后必须重启服务才能在搜索中生效
2. **索引不存在会 503**：`CrmEnum.initIndex=false` 的模块（如 customer/contract）若 ES 中无对应索引，前端列表查询会报错
3. **冗余字段异步更新有短暂不一致**：`ElasticUtil.batchUpdateEsData` 是异步执行，改名后列表页名称可能有几秒延迟
4. **bulk 操作无事务保证**：ES 批量写入失败仅重试1次，大量失败时数据可能丢失
5. **EsUtil.getClient() 有竞态风险**：双重检查锁定但 `client` 字段非 `volatile`（实际影响极低）

### 11.2 性能优化建议

| 优化项 | 建议 |
|--------|------|
| JVM 堆内存 | 生产环境建议 ≥2g，当前 Docker 配置为 1g |
| 分片策略 | 小索引（经销商退网等）可降为 1 分片，减少资源占用 |
| 初始化速度 | 全量同步时可临时设 `refresh_interval=-1`，完成后恢复 |
| bulk 批量大小 | 当前 5000 条/批，文档较大时可适当减小 |
| 连接池 | `connectionCheckEnabled=false`，建议生产开启 |
| mapping 缓存 | 建议增加手动刷新接口，避免新增字段后必须重启 |

### 11.3 与 Nacos/Redis 的协同

- **Nacos**：管理 `wk-es.yml` 连接参数，各模块统一引用
- **Redis**：缓存用户信息（`UserCacheUtil`）、字段配置（`queryFieldByCache`），辅助 ES 查询时补充名称等冗余数据
- **MySQL**：主存储，ES 为搜索加速层，两者通过双写保持一致性（最终一致）

---

## 12. MySQL 与 ES 数据一致性机制

### 12.1 整体架构：同步双写 + 异步级联

```
MySQL（主存储，事务保证）
    │
    ├── 同步双写 ──→ ES（搜索加速层）
    │   ├── savePage()     → EsUtil.updateField(docAsUpsert=true)
    │   ├── updateField()  → EsUtil.updateField()
    │   └── deletePage()   → EsUtil.deleteData()
    │
    └── 异步级联 ──→ ES（冗余名称字段）
        └── ElasticUtil.batchUpdateEsData() → ThreadPoolTaskExecutor
```

**核心原则**：MySQL 是主存储（强一致），ES 是搜索加速层（最终一致）。列表查询全部走 ES，详情查询走 MySQL。

### 12.2 同步双写模式（主流程）

所有 CRM 业务 ServiceImpl 遵循统一的写入顺序：

**保存流程（以 CrmJtContractChangeServiceImpl 为例）**：
```
1. @Transactional 开启 MySQL 事务
2. updateById(entity)             ← 先写 MySQL
3. fieldDataService.saveData()    ← 写自定义字段数据
4. savePage(crmModel, dataId)     ← 同步写 ES（调用链如下）
     └→ CrmPageService.savePage()
         └→ FieldPageService.savePage()
             └→ EsUtil.updateField(map, id, label, docAsUpsert=true)
5. MySQL 事务提交
```

**删除流程（以 CrmJtContractChangeServiceImpl.deleteByIds 为例）**：
```
1. @Transactional 开启 MySQL 事务
2. lambdaUpdate().set(deleted, 1) ← 先 MySQL 逻辑删除
3. deletePage(ids)                ← 同步删 ES
     └→ FieldPageService.deletePage()
         └→ EsUtil.deleteData(ids, getLabel())
4. MySQL 事务提交
```

**负责人变更流程**：
```
1. MySQL updateById               ← 先更新 MySQL
2. updateField(map, ids)          ← 同步更新 ES（含 ownerDeptId/ownerDeptName）
```

### 12.3 ES 操作的容错与重试机制

| 操作 | 方法 | 重试策略 | 失败处理 |
|------|------|---------|---------|
| 单条 update | `EsUtil.updateField()` | `retryOnConflict(3)`（乐观锁冲突重试3次） | 404 静默忽略，其他异常抛出 `CrmException` |
| 批量 bulk | `EsUtil.bulk()` | 失败文档重试 1 次（仅重试失败 ID） | 超过重试次数后静默丢弃（仅日志） |
| 单条 delete | `EsUtil.deleteData()` | 无重试 | 异常仅记录日志 |
| docAsUpsert | `updateField(docAsUpsert=true)` | — | 文档不存在时自动创建，避免不一致 |

### 12.4 docAsUpsert 策略详解

`savePage()` 最终调用 `EsUtil.updateField(map, id, label, true, isRefreshIndex)`，其中 `docAsUpsert=true`：

- **文档已存在**：执行部分更新（仅更新 map 中的字段）
- **文档不存在**：自动创建新文档（防止 MySQL 有数据但 ES 无文档的不一致）

这一策略是关键的安全网——即使因 ES 临时不可用导致某次写入失败，下一次 savePage 调用也能自动补齐 ES 文档。

### 12.5 异步冗余字段级联（短暂不一致窗口）

`ElasticUtil.batchUpdateEsData()` 使用 `ThreadPoolTaskExecutor` 异步执行：

```
用户改名 → ElasticUtil.batchUpdateEsData("user", userId, newName, companyId)
    └→ 线程池异步执行：
        1. 查 MySQL 找到所有 owner_user_id = userId 的记录 ID
        2. 批量调用 EsUtil.updateField() 更新 ES 中的 ownerUserName
```

**不一致窗口**：从 MySQL 改名完成到 ES 更新完毕，通常 **数百毫秒到数秒**，取决于数据量。

**代码中的特殊处理**（解决同时异步更新时的版本冲突）：
```java
// 将用户更新分三批执行，避免 owner 和 creator 同时更新引发乐观锁冲突
// 批次1：owner_user_id = id AND create_user_id != id
// 批次2：create_user_id = id AND owner_user_id != id
// 批次3：owner_user_id = id AND create_user_id = id（同时更新两个字段）
```

### 12.6 事务边界与一致性风险

```
@Transactional(rollbackFor = Exception.class)  ← 仅管 MySQL
public List<OperationLog> deleteByIds(List<Long> ids) {
    lambdaUpdate().set(deleted, 1).in(...).update();  // MySQL 事务内
    deletePage(ids);                                   // ES 操作，不在事务内！
    return operationLogList;
}
```

**关键问题：ES 操作不在 MySQL 事务管理范围内**。可能出现的异常场景：

| 场景 | MySQL 状态 | ES 状态 | 结果 |
|------|-----------|---------|------|
| MySQL 成功 + ES 成功 | 已删除 | 已删除 | 正常 |
| MySQL 成功 + ES 失败 | 已删除 | 未删除 | 列表仍显示已删数据（脏数据） |
| MySQL 回滚 + ES 成功 | 未删除 | 已删除 | 列表中丢失数据 |
| MySQL 回滚 + ES 失败 | 未删除 | 未删除 | 正常（但 ES 操作无回滚机制） |

### 12.7 当前项目的实际一致性保证等级

| 操作类型 | 一致性等级 | 说明 |
|---------|-----------|------|
| 新建/编辑保存 | **准强一致** | 同步双写 + docAsUpsert 兜底，ES 写入失败不影响 MySQL |
| 删除 | **最终一致（弱）** | ES 删除无重试，失败概率较高时可能产生脏数据 |
| 负责人变更 | **准强一致** | 同步 updateField + retryOnConflict(3) |
| 冗余名称字段 | **最终一致（秒级）** | 异步线程池更新，有短暂不一致窗口 |
| 全量初始化 | **最终一致** | 服务启动时 50 线程并发同步，完成后一致 |

### 12.8 现有补偿机制

1. **服务重启全量同步**：`InitEsIndexRunner.initData()` 在服务启动时从 MySQL 全量同步数据到 ES，是最终的兜底机制
2. **docAsUpsert 自愈**：下次 savePage 时自动创建缺失的 ES 文档
3. **refresh 策略**：关键写入设置 `Refresh.WaitFor` 或 `Refresh.True`，确保写入后立即可搜索

### 12.9 潜在改进方向

| 方向 | 方案 | 说明 |
|------|------|------|
| 删除重试 | `deleteData` 增加重试逻辑 | 当前删除无重试，网络抖动可能导致 ES 残留脏数据 |
| 事务补偿 | 监听 MySQL 事务提交结果，失败时回滚 ES | 需引入 TransactionSynchronization |
| 消息队列 | 用 MQ 异步消费 MySQL binlog | 代码注释中已标注 `todo 暂时保留，冗余数据以后通过 mq 更新` |
| 定时对账 | 定期比对 MySQL 和 ES 数据量/ID 差异 | 可发现并修复长期积累的不一致 |
| ES 操作日志 | 记录 ES 写入失败到补偿表，定时重试 | 防止 ES 临时不可用时的数据丢失 |

---

## 13. 关键文件路径汇总

| 文件 | 说明 |
|------|------|
| `common/common-field/pom.xml` | ES 依赖声明 |
| `common/common-field/.../config/ElasticClientProperties.java` | 配置属性类 |
| `common/common-field/.../config/FieldAutoConfiguration.java` | 客户端 Bean 自动配置 |
| `common/common-field/.../constant/EsFieldTypeEnum.java` | ES 字段类型枚举 |
| `common/common-field/.../utils/EsUtil.java` | ES 操作工具类（2051 行） |
| `common/common-field/src/main/resources/wk-es.yml` | ES 连接配置 |
| `crm/crm-common/.../constant/CrmEnum.java` | 业务枚举（控制索引名/initIndex） |
| `crm/crm-web/.../common/InitEsIndexRunner.java` | CRM 索引初始化（1156 行） |
| `crm/crm-web/.../common/ElasticUtil.java` | 冗余字段级联更新 |
| `crm/crm-web/.../service/CrmPageService.java` | 列表搜索标准接口（1342 行） |
| `email/email-web/.../common/InitEsIndexRunner.java` | Email 索引初始化 |
| `km/km-web/.../common/InitEsIndexRunner.java` | KM 索引初始化 |
| `docker/es-docker-compose.yml` | ES Docker 部署配置 |
| `docker/es/Dockerfile` | ES 镜像构建（含插件） |
