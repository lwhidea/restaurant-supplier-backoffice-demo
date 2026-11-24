# 餐饮供应商服务后台技术方案 v2.28

更新时间：2026-11-23  
适用范围：身份工作区和授权、跨领域模块承接、固定/按日需求、投标定标、订单履约与追加、收货结算，以及总部订单聚合和临时采购批量归档。  
关联 PRD：[餐饮供应商服务后台-PRD与线上开发计划-v2.28.md](餐饮供应商服务后台-PRD与线上开发计划-v2.28.md)

## 1. 设计原则

- 聚合只改变列表查询和展示，不改变订单、订单行、临采记录的事实边界。
- 展示摘要由业务字段计算，不将摘要字符串写回订单作为数据源。
- 单笔和批量归档最终调用同一临采归档领域逻辑，避免两套状态更新规则。
- 批量归档必须原子提交；请求重放不会重复创建归档副作用。
- 保留历史记录与审计链；新增字段可选，兼容现有 v2.22 demo 的数组型订单与 localStorage 数据。

## 2. 总部订单聚合

### 2.1 读取模型

线上订单事实仍以 `purchase_order` 与 `purchase_order_line` 为准。列表 API 可提供门店聚合视图：

```json
{
  "storeId": "store_123",
  "storeName": "车城店",
  "orderCount": 4,
  "supplierCount": 2,
  "categoryCount": 3,
  "skuCount": 5,
  "deliveryWindows": ["2026-11-23T08:00:00+08:00"],
  "aggregateStatus": "PARTIALLY_INVALID",
  "statusCounts": {"ACTIVE": 3, "INVALID": 1},
  "childrenCursor": null
}
```

摘要口径：

- 供应商数：该门店当前查询结果内按 `supplier_id` 去重。
- 品类数：该门店子订单行内按 `category_id` 去重。
- SKU 数：按 `sku_id` 去重；品类数与 SKU 数并列展示为“X 个品类 · Y 个 SKU”。
- 状态：`PARTIALLY_INVALID` 表示失效/取消与其他状态并存；`ALL_INVALID` 表示所有子订单均失效或取消。所有有效订单状态相同则展示该状态；混合有效状态显示 `PROCESSING`。
- 摘要与明细使用同一筛选条件、授权范围和一致性快照，展开时不得返回其他门店数据。

### 2.2 接口建议

`GET /v1/orders?groupBy=store&status=&keyword=&cursor=&limit=` 返回聚合行和分页游标。展开通过 `GET /v1/orders?storeId={id}&cursor=&limit=` 读取子订单。若列表规模较小，也可通过 `include=children` 一次返回；前端统一使用 `supplierCount/categoryCount/skuCount/aggregateStatus`，不依赖具体加载方式。

子订单继续引用现有 `Order.v1`：订单号、供应商、门店、配送窗口、状态和订单行。固定需求与按日计划都使用同一订单读取模型，配送日为空时显示固定需求到货窗口。

### 2.3 状态聚合伪逻辑

```text
invalid = child.status in {INVALID, CANCELLED}
if invalidCount == childCount: ALL_INVALID
else if invalidCount > 0: PARTIALLY_INVALID
else if distinct(active statuses) == 1: that status
else: PROCESSING
```

订单级 `INVALID` 与供应商违约的关系需在服务端状态机中映射；取消记录如果不应计作失效，应返回独立 `CANCELLED` 计数并由产品口径明确。当前 demo 将已取消纳入“失效/取消”聚合；线上应统一机器状态码后再落库。

## 3. 临时采购 SKU 批量归档

### 3.1 数据事实

每笔临采来源登记对应一个 `temporary_procurement` 主记录，关联一笔或多笔来源缺口行；每条缺口行仍保留 `store_id`、`order_id`、`sku_id`、数量、原供应商、原违约订单及其处理状态。按 SKU 的分组是操作视图，不将多门店、多订单的数量事实合成一条记录。

建议归档字段：`source_supplier_name` 或 `source_supplier_id`、`unit_price_minor`、`currency`、`note`、`archived_by`、`archived_at`、`version`、`idempotency_key`。来源供应商尚未进入平台供应商主档时，允许有审计的临时来源文本；后续供应商身份确认通过映射关联，不覆盖原始来源快照。

### 3.2 批量接口

```http
POST /v1/temp-procurements/batch-archive
Idempotency-Key: <unique-request-key>
```

```json
{
  "items": [
    {
      "skuId": "sku_789",
      "sourceSupplierName": "本地临采供应商",
      "unitPriceMinor": 12800,
      "note": "原中标供应商未按期供货",
      "caseIds": ["case_1", "case_2"]
    }
  ]
}
```

服务端逐项检查：调用者具备管理员权限；所有 case 处于 `READY_FOR_ARCHIVE`；case 的 SKU 与请求 SKU 一致；记录属于当前租户；价格为非负整数分且单位一致；来源及备注符合必填规则；没有被其他操作并发归档。全部校验通过后，在同一数据库事务中归档每条 case、写操作审计和 outbox 事件；任一校验失败则整批回滚并返回逐条错误。相同幂等键重放返回首次结果。

单笔归档应复用同一个领域服务，例如 `archiveTemporaryProcurement(caseId, source, actor, idempotencyKey)`；批量 API 在单事务中调用该服务的事务内变体。

## 4. Demo 实现约束

- 继续使用原 localStorage 键 `supply-prototype-data-v222`，本次只新增页面内展开状态与批量表单交互，不改动持久化键、不清空旧数据。
- 总部订单以门店为父行；展开显示订单号、供应商、SKU 需求、到货窗口及原跟踪入口。
- 批量归档按选中的 case 组装 SKU 表单；一个 SKU 填一次来源、单价、备注。只允许选中 `待管理员归档`，其余步骤仍走现有“临采已安排”流程。
- 批量提交前一次性检查所有 SKU 表单，全部通过才更新 case 与对应原订单；保留单笔归档能力。
- 旧版订单字段仍可为数组，读取适配器从订单明细、SKU 主档与招标计划逐级解析；无法解析的历史商品显示“未分类 / 商品信息待补”，不可伪造 SKU 统计。

## 5. 开发顺序与验收点

1. 在 `packages/contracts` 定义聚合 DTO、状态枚举、批量请求和逐条错误响应。
2. 在订单查询服务实现租户/门店范围过滤后聚合，并提供一致的子订单游标查询。
3. 在临采领域服务实现单笔归档复用和原子批量归档，添加唯一幂等记录及审计/outbox 写入。
4. 前端增加门店折叠行、摘要、子订单列表和 SKU 批量登记表单；页面授权只控制操作显示，后端再次校验。
5. 数据库迁移采用可回滚、向后兼容方式；先部署新增字段与接口，再部署前端，旧客户端继续使用单笔 API。

验收要求：多供应商同店订单供应商数不重复；品类与 SKU 计数正确；部分/全部失效状态准确；展开明细与父行汇总一致；跨门店无数据泄露；批量选择跨门店同 SKU 能按 SKU 统一填来源但仍逐 case 归档；混选不同 SKU 可分别填来源；任一字段错误时整批不落库；并发单笔/批量归档只允许一个操作成功；老记录刷新后仍可读且没有被重写。

## 6. 多供应商公平分配与供量校验

### 6.1 分配算法

以每个 `planId + skuId` 为计算范围，先按已选供应商形成门店分配组。每组可分配池为该供应商该 SKU 的申报可供量，扣除管理员手动锁定的门店数量后，在未锁定门店间做 max-min 均分：

1. 将剩余供量按当前未满足门店数平均分配；单店先取 `min(均分额, 剩余需求)`。
2. 达到需求上限的门店退出分配组，其未使用配额重新均分给其他门店。
3. 直到该供应商可供量用尽或该组门店需求全部满足。
4. 用户把某门店改派给其他供应商后，重算旧供应商剩余未锁定门店组与新供应商分组；保留其他已锁定数量。供应商能力变化或门店需求快照更新时，重新生成分配草稿并要求管理员确认。
5. 数量按 SKU 计量步进转为整数 tick 运算，避免浮点误差；包装类通常步进 1，可称重类按主档允许步进。不能整除的余量按稳定的门店排序每次补一个 tick。

### 6.2 数量合计和操作保护

- 对每个供应商 SKU 维护 `allocatedQty = sum(storeAllocation.qty)`，显示 `allocatedQty / supplierAvailableQty` 与未分配余量；不要求强行用满供应量，需求较少时允许余量保留。
- 每个门店数量字段动态上限为 `min(storeNeed, supplierAvailableQty - sameSupplierOtherStoreAllocations)`；更改提交时服务端重新计算并校验，浏览器 `max` 属性不作为安全边界。
- 若用户将当前门店数调高，超过供量则拒绝本次修改并解释“该供应商 SKU 分配合计不能超过可供量”；可先下调其他门店再调整。 历史未完成记录若固定量合计已超过能力上限，重入定标时解除该组冲突锁定并按剩余需求均分；已生成订单不自动修改。
- 关键数值（供应商可供量、已分配量、剩余量、未覆盖需求）以固定标签与高对比数值展示；颜色同时配合文本状态，不能仅靠颜色表达。
- 定标事务需锁住对应的供应商 SKU 容量行或采用版本条件更新，防止并发定标/修改把供量重复承诺。

## 7. 身份工作区与领域模块交接

### 7.1 角色授权模型

每个请求建立服务端授权上下文：`tenantId + actorId + roleCodes[] + resourceScope + action`。资源范围可为租户、绑定门店或供应商主体。前端收到可用模块清单仅用于导航和按钮呈现；列表查询、详情读取、写入命令、下载附件均在服务端再次校验范围。多角色用户每次命令需带当前岗位上下文，但后端仍验证该岗位真实绑定关系。

建议把身份与业务权限分离存储：`role_definition` 存角色模板；`permission` 表示资源/动作能力；`role_permission` 做授权组合；`membership` 连接用户与租户；`role_binding` 连接 membership 与门店或供应商主体；需要临时跨店管理时使用带有效期、审批人和原因的 scope grant。拒绝访问应返回稳定错误码，不依赖 UI 隐藏控制。

### 7.2 模块所有权与接口方向

| 模块 | 写入所有者 | 主要命令 / 查询边界 | 向下游发布的事实 |
| --- | --- | --- | --- |
| Identity & Organization | 身份/组织域 | 角色绑定、门店/供应商授权、主数据维护；统一授权查询 | `MembershipChanged`、`RoleScopeChanged`、主数据版本事件 |
| Plan & Demand | 计划/需求域 | 创建计划、提交需求、审核、发布快照；策略适配器把固定模式和按日模式转换成相同 `DemandSnapshot` | `DemandSnapshotPublished`、`DemandSnapshotSuperseded` |
| Sourcing & Award | 报价/定标域 | 报价、能力申报、定标模拟、分配确认；只读消费已发布快照 | `QuoteSubmitted`、`AwardConfirmed`、`AwardGapDetected` |
| Order & Fulfillment | 订单履约域 | 基准订单生成、接单、拒接、发货、追加派单和兜底改派 | `OrderIssued`、`OrderAccepted`、`ShipmentReported`、`AdditionAccepted` |
| Receiving | 收货验收域 | 收货单读模型、验收提交、更正/冲正 | `ReceiptAccepted`、`ReceiptCorrectionRequested` |
| Finance | 财务域 | 应付生成、发票占用/核验、付款记录、归档 | `PayableReady`、`InvoiceVerified`、`PaymentRecorded`、`SettlementArchived` |
| Exception Procurement | 异常采购域 | 临采建单、来源批量归档、违约处置 | `TemporaryProcurementArchived`、`SupplierViolationRecorded` |
| Workflow & Analytics | 平台支撑域 | 事件订阅派发待办；构建只读聚合投影 | 待办/报表投影；不拥有交易状态写入权 |

页面模块可以各自部署，但命令必须落到单一领域所有者。跨模块不得共享可写 ORM 实体或直接更新他域数据表。读侧可使用 API 组合或异步投影；需要即时强一致的校验由写入所有者完成。

### 7.3 标准跨模块流程

1. 组织域确认用户角色与资源范围后，计划域允许管理员建立固定或按日计划。
2. 门店需求提交经需求域校验；管理员发布后生成不可变快照，事件通知供应商端和定标待办。
3. 报价域只接受快照版本对应的供应商报价。定标域按 SKU、门店、日期及供应商能力校验后，在一次事务中锁定分配版本并写 outbox 事件。
4. 订单域消费确认的分配，只生成一次标准订单；按供应商 × 门店 × 日期聚合多 SKU 行。供应商接单与管理员超时兜底通过版本条件更新竞争同一订单承诺。
5. 追加申请先进入需求域；审核/定向派单由订单履约域承接。供应商接单成功后写追加订单行，沿用原价；拒绝或过期事件推动兜底待办。
6. 收货验收域按订单行接收门店实收与判定，确认的合格量生成财务域应付行。开票按门店 × 日期 × 供应商 × 一级分类分组，不等待月账期；每一应付行通过唯一占用约束防止重复开票。
7. 违约或供量缺口进入异常采购域，临采单独留存来源并以正常订单/验收/结算契约回接下游，原供应商订单不被覆盖。

### 7.4 状态机与竞争保护

数据库保存机器状态码，页面文案只做本地化映射。命令需提交 `expectedVersion`；服务端验证当前状态允许转换后，以条件更新递增聚合版本。关键转换的写所有者：计划/快照归计划需求域，报价归寻源域，定标分配归定标域，接单/发货/追加归订单履约域，实收归收货域，应付/发票/付款归财务域。管理员不能通过角色提升绕过各域业务门槛。

状态迁移表应登记 `fromState/toState/command/actorScope/preconditions/sideEffects/rejectionCode`；初始化就建立迁移白名单。自动超时任务只能执行已登记转换并持幂等任务键；退回和冲正必须形成新版本/新记录。状态码改义或删除须发布契约主版本。

### 7.5 数据库约束、索引与租户隔离

- 每张交易表有 `tenant_id`；对同租户关联使用复合外键或服务端约束验证跨租户引用。优先采用强制 tenant-scoped repository；若所选 PostgreSQL 托管环境支持且团队验证可靠，可再加 RLS 作为纵深防护，不能以 RLS 代替应用授权。
- 所有租户内业务号有唯一键；`(tenant_id, plan_id, snapshot_revision)` 唯一；供应商 × 快照版本 × SKU 报价唯一；日能力按 quote line × delivery date 唯一。
- 分配行唯一粒度含定标版本、供应商、门店、SKU、配送日和来源需求行；基准订单生成唯一粒度按分配版本 × 供应商 × 门店 × 日期；追加订单以追加申请行作唯一来源键。
- 应付行以验收来源行唯一；有效发票通过关联唯一约束占用应付行；付款分摊不得大于发票未付余额。撤票、冲正和重开只能追加状态/关联记录，不能删除历史。
- `idempotency_record` 按租户、调用者/接口范围和幂等键唯一；`consumer_inbox` 按消费者和 event ID 唯一；workflow task 按业务实体 × 任务类型 × 责任 scope 去重。
- 数量按单位精度保存 `numeric` 或整数 tick，并由 SKU step 校验；价格/金额使用最小货币单位整数；检查数量非负、合格量不大于实收量、实收不超过订单剩余量。停用主数据保留引用，不物理删除交易行。
- 关键复合索引覆盖 tenant + store/supplier + deliveryDate + status，plan + snapshot revision，supplier + tender + status，workflow assignee scope + dueAt + status；上线前基于代表性查询计划检查索引。

### 7.6 数据导入、附件与日期规则

demo 的 localStorage 和浏览器文件名均不是生产数据源；默认不导入。真实旧系统数据如需导入，应另立经批准批次，执行只读导出、稳定 ID 映射、重复/孤儿报告、dry-run 数量金额对账、签字确认、批次隔离和回滚。迁移任务可重跑但不得重复创建交易；导入源 ID 保留追溯。

配送日按租户时区的本地日历日期保存；具体时刻保存 UTC 同时带 IANA 时区。D-1 使用前一个本地自然日，不自动跨节假日顺延；DST 由时区数据库转换。文件上传先到隔离区，经大小/MIME/恶意内容扫描后才能关联发票、验收或订单；下载链接短时签名且服务端再次校验实体权限。保留/删除期限需由合规和财务负责人定值。恢复验收包含数据库、对象、outbox/inbox 和审计流水的引用一致性。

### 7.7 事件信封、幂等与失败恢复

领域事件统一信封至少包含 `eventId`、`eventType`、`schemaVersion`、`tenantId`、`aggregateType`、`aggregateId`、`aggregateVersion`、`occurredAt`、`actorId`、`correlationId`、`causationId` 与 `payload`。payload 仅含下游完成业务所需字段，不传播超出接收者授权范围的数据。附件使用有时效、按业务授权校验的对象引用，不把公开 URL 写入事件。

同一数据库事务写业务状态和 outbox；消费者以 `eventId` 幂等，允许至少一次投递。跨域流程不依赖消息严格顺序：消费者按 `aggregateVersion` 拒绝旧版本、延迟版本可补读权威 API。消息重试、死信和人工重放均保留审计记录。对需要竞争的接单/兜底、发票占用、追加接单采取条件更新或唯一约束；对发票上传、文件生成等外部副作用用可重试的状态机。



## 8. 尚需实现前确认的接口细节

1. 若同 SKU 的跨门店临采实际来自不同供应商，批量表单允许按 SKU 统一来源就意味着该组共用一个来源；按当前已确认规则，单次批量操作按 SKU 统一来源；如果真实临采同时有多个来源，用户应拆成多次操作，或在后续版本将分组键升级为“SKU × 来源供应商”。
2. 取消订单是否计入“全部失效”仍需统一 API 状态语义。当前产品文案把取消与失效都视为不再有效；服务端应保留独立原状态及分项计数供审计。
