# ODS 层（原始数据层）开发规则

## 1. 概述

ODS（Operational Data Store，原始数据层）是数仓体系中的**最底层**，承担着将各业务系统、外部数据源的原始数据接入数仓的职责。ODS 层的核心原则是**保持源数据结构不变、数据不丢失**，为上层 DWD 清洗加工提供可靠的数据基础。本文档定义了 ODS 层的开发规范，确保数据接入的完整性、时效性和可追溯性。

---

## 2. 分层定义

### 2.1 ODS 层定位

- **定位**: 业务系统原始数据的镜像层，保持与源系统一致的数据结构和数据内容。
- **核心职责**:
  - 业务数据的统一接入与落地
  - 保持源数据结构完整，不做业务逻辑加工
  - 提供数据的可追溯性和历史回溯能力
  - 作为 DWD 层加工的唯一数据源
- **设计原则**:
  - **原样接入**: 不对数据做清洗、转换、聚合等操作
  - **全量保留**: 源表所有字段全部接入，不做裁剪
  - **可追溯**: 保留数据接入时间、来源等元数据信息

### 2.2 ODS 层在数仓中的位置

```
┌──────────────────────────────────────────────────┐
│              上层消费（DWD / DWM / DM / APP）       │
├──────────────────────────────────────────────────┤
│                   ODS 层（原始数据层）               │
│          原始数据接入，保持源数据结构不变               │
├──────────────────────────────────────────────────┤
│                   数据源（业务系统）                  │
│      MySQL / Oracle / MongoDB / Kafka / 日志 / API │
└──────────────────────────────────────────────────┘
```

---

## 3. 命名规范

### 3.1 库名规范

```
格式: ods_{数据来源/业务系统}
示例:
  - ods_trade         -- 交易系统
  - ods_user          -- 用户系统
  - ods_payment       -- 支付系统
  - ods_log           -- 日志数据
  - ods_external      -- 外部数据
```

### 3.2 表名规范

```
格式: ods_{来源系统}_{源表名}_{更新周期}
示例:
  - ods_trade_orders_di              -- 交易系统-订单表-日增量
  - ods_trade_orders_df              -- 交易系统-订单表-日全量
  - ods_user_user_info_df            -- 用户系统-用户信息表-日全量
  - ods_payment_pay_record_di        -- 支付系统-支付记录表-日增量
  - ods_log_app_click_hi             -- 日志-APP点击日志-小时增量
  - ods_external_weather_data_df     -- 外部数据-天气数据-日全量
```

### 3.3 字段命名规范

| 规则 | 说明 | 示例 |
|------|------|------|
| 保持源字段名 | ODS 层字段名与源系统保持一致 | 源表为 `orderNo` 则保留 `orderno` |
| 统一小写 | 所有字段名统一转为小写 | `OrderId` → `orderid` |
| 元数据字段 | 以 `ods_` 前缀标识 ODS 层新增的元数据字段 | `ods_create_time`, `ods_source` |

> **注意**: ODS 层不对字段进行重命名或语义转换，字段含义的标准化在 DWD 层完成。

---

## 4. 建表规范

### 4.1 DDL 模板

```sql
-- ============================================================
-- 表名: {表名}
-- 描述: {源系统}.{源表名} 的 ODS 镜像表
-- 负责人: {负责人}
-- 创建日期: {YYYY-MM-DD}
-- 更新周期: {di/df/hi}
-- 数据来源: {源系统名称}.{源库名}.{源表名}
-- 同步方式: {DataX/Canal/Sqoop/Flume/Kafka}
-- ============================================================

CREATE TABLE IF NOT EXISTS {db_name}.{table_name} (
    -- ========== 源系统字段（保持原样） ==========
    orderid             STRING          COMMENT '订单ID（源字段: OrderId）',
    userid              BIGINT          COMMENT '用户ID（源字段: UserId）',
    order_amount        DECIMAL(18,2)   COMMENT '订单金额（源字段: OrderAmount）',
    order_status        STRING          COMMENT '订单状态（源字段: OrderStatus）',
    create_time         STRING          COMMENT '创建时间（源字段: CreateTime）',
    update_time         STRING          COMMENT '更新时间（源字段: UpdateTime）',
    
    -- ========== ODS 元数据字段 ==========
    ods_create_time     STRING          COMMENT 'ODS 数据接入时间',
    ods_source          STRING          COMMENT '数据来源标识'
)
COMMENT '{源系统名称}.{源表名} 原始数据镜像'
PARTITIONED BY (
    dt                  STRING          COMMENT '日期分区, 格式: YYYYMMDD'
)
STORED AS ORC
TBLPROPERTIES (
    'orc.compress' = 'SNAPPY'
);
```

### 4.2 必填元数据字段

每张 ODS 表**必须**包含以下元数据字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `ods_create_time` | STRING | ODS 数据接入时间，格式 `yyyy-MM-dd HH:mm:ss` |
| `ods_source` | STRING | 数据来源标识，如 `trade_db.orders` |
| `dt` | STRING | 日期分区字段（分区键），格式 `YYYYMMDD` |

### 4.3 数据存储规范

- **存储格式**: 统一使用 **ORC** 格式
- **压缩方式**: 统一使用 **SNAPPY** 压缩
- **分区策略**:
  - 必须按日期 `dt` 进行分区
  - 日志类数据量大时，增加小时分区 `hour`
  - 分区字段不允许出现在表字段中
- **数据保留**:
  - ODS 层数据默认保留 **90 天**
  - 重要业务数据（交易、财务）保留 **1 年**
  - 日志类数据保留 **30 天**（可根据存储成本调整）

---

## 5. 数据接入规范

### 5.1 接入方式选择

| 源数据类型 | 推荐工具 | 同步方式 | 适用场景 |
|-----------|----------|----------|---------|
| MySQL / PostgreSQL | DataX / Sqoop | 批量同步 | 定时全量/增量同步 |
| MySQL Binlog | Canal / Maxwell | 实时同步 | 需要实时或准实时的场景 |
| MongoDB | DataX (MongoDB Reader) | 批量同步 | 文档型数据库同步 |
| Kafka / 消息队列 | Flume / Spark Streaming | 流式接入 | 日志、埋点、事件流 |
| 日志文件 | Flume / Logstash | 文件采集 | 服务器日志、应用日志 |
| API / HTTP | 自定义脚本 | 定时拉取 | 第三方数据接入 |
| FTP / SFTP | DataX (FTP Reader) | 文件传输 | 外部合作方数据 |

### 5.2 全量同步规范

```sql
-- ============================================================
-- 任务名: ods_user_user_info_df
-- 描述: 用户信息表全量同步
-- 同步工具: DataX
-- 调度周期: 每日 01:00
-- ============================================================

-- 全量同步采用 INSERT OVERWRITE 方式
INSERT OVERWRITE TABLE ods_user.ods_user_user_info_df PARTITION(dt = '${bizdate}')
SELECT
    -- 源系统字段（全量保留）
    user_id,
    user_name,
    phone,
    email,
    register_time,
    user_status,
    
    -- ODS 元数据
    CURRENT_TIMESTAMP()     AS ods_create_time,
    'user_db.user_info'     AS ods_source

FROM source_data_staging_table
;
```

### 5.3 增量同步规范

```sql
-- ============================================================
-- 任务名: ods_trade_orders_di
-- 描述: 订单表增量同步
-- 同步工具: DataX
-- 调度周期: 每日 02:00
-- 增量策略: 基于 update_time 抽取当日变更数据
-- ============================================================

-- 增量同步抽取条件
-- WHERE update_time >= '${bizdate} 00:00:00'
--   AND update_time <  '${bizdate+1} 00:00:00'

INSERT OVERWRITE TABLE ods_trade.ods_trade_orders_di PARTITION(dt = '${bizdate}')
SELECT
    -- 源系统字段
    order_id,
    user_id,
    order_amount,
    order_status,
    create_time,
    update_time,
    
    -- ODS 元数据
    CURRENT_TIMESTAMP()         AS ods_create_time,
    'trade_db.orders'           AS ods_source

FROM source_data_staging_table
WHERE update_time >= '${bizdate} 00:00:00'
  AND update_time <  DATE_ADD('${bizdate}', 1) || ' 00:00:00'
;
```

### 5.4 增量字段选择原则

| 增量字段类型 | 适用场景 | 注意事项 |
|-------------|---------|---------|
| `update_time` | 有更新时间戳的业务表 | 最常用，需确认源表该字段可靠 |
| `create_time` | 只新增不更新的流水表 | 需确认无更新场景 |
| 自增 ID | 严格自增的流水类数据 | 需确认 ID 严格递增 |
| Binlog 位点 | 实时同步场景 | 需维护位点信息 |

---

## 6. 数据质量规范

### 6.1 必配质量检查

每个 ODS 接入任务**必须**包含以下质量检查：

| 检查项 | 说明 | 必选 | 告警阈值 |
|--------|------|------|---------|
| 接入行数检查 | 确保数据量在合理范围内 | ✅ | 行数为 0 或波动 > 50% |
| 源表行数对比 | ODS 表行数与源表行数对比 | ✅ | 差异 > 1% |
| 主键唯一性 | 全量表主键不重复 | ✅ | 重复率 > 0 |
| 空值比例 | 核心字段空值率 | ✅ | 空值率 > 阈值 |
| 时效性检查 | 最新数据时间是否符合预期 | ⬜ | 最新记录超过 24h |
| 字段长度检查 | 字段值长度是否在合理范围 | ⬜ | 超长截断告警 |

### 6.2 质量校验 SQL 示例

```sql
-- 接入行数检查 & 波动检查
SELECT
    today.cnt       AS today_cnt,
    yesterday.cnt   AS yesterday_cnt,
    CASE
        WHEN yesterday.cnt > 0
        THEN ABS(today.cnt - yesterday.cnt) / yesterday.cnt
        ELSE NULL
    END             AS fluctuation_rate
FROM (
    SELECT COUNT(1) AS cnt FROM ods_trade.ods_trade_orders_di WHERE dt = '${bizdate}'
) today
CROSS JOIN (
    SELECT COUNT(1) AS cnt FROM ods_trade.ods_trade_orders_di WHERE dt = '${yesterday}'
) yesterday
;

-- 主键唯一性检查（全量表）
SELECT
    COUNT(1) AS total_cnt,
    COUNT(DISTINCT order_id) AS distinct_cnt
FROM ods_trade.ods_trade_orders_df
WHERE dt = '${bizdate}'
HAVING COUNT(1) <> COUNT(DISTINCT order_id)
;

-- 空值检查
SELECT
    COUNT(1) AS total_cnt,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS order_id_null_cnt,
    SUM(CASE WHEN user_id IS NULL THEN 1 ELSE 0 END)  AS user_id_null_cnt,
    SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) / COUNT(1) AS order_id_null_rate,
    SUM(CASE WHEN user_id IS NULL THEN 1 ELSE 0 END)  / COUNT(1) AS user_id_null_rate
FROM ods_trade.ods_trade_orders_di
WHERE dt = '${bizdate}'
;

-- 时效性检查（检查最新数据时间）
SELECT
    MAX(update_time) AS latest_update_time
FROM ods_trade.ods_trade_orders_di
WHERE dt = '${bizdate}'
HAVING MAX(update_time) < DATE_SUB('${bizdate}', 1)
;
```

---

## 7. 性能优化规范

### 7.1 同步性能优化

| 优化项 | 说明 |
|--------|------|
| 并行度控制 | DataX/Sqoop 的并行通道数根据源库承受能力调整，建议 3-8 个 |
| 分批同步 | 大表（> 1亿行）建议按主键范围分批同步 |
| 网络优化 | 同步任务部署在与源库同机房/同区域的节点 |
| 错峰调度 | 避开业务高峰期（如大促期间），错峰执行同步任务 |
| 增量优先 | 大表优先使用增量同步，减少数据传输量 |

### 7.2 存储优化

| 优化项 | 说明 |
|--------|------|
| 生命周期管理 | 配置分区过期自动清理策略 |
| 合并小文件 | 接入完成后合并小文件，减少 NameNode 压力 |
| 合理分区 | 避免过多分区（单表分区数建议 < 10000） |

### 7.3 小文件合并配置

```sql
-- 同步完成后合并小文件
SET hive.merge.mapfiles = true;
SET hive.merge.mapredfiles = true;
SET hive.merge.size.per.task = 256000000;
SET hive.merge.smallfiles.avgsize = 128000000;
```

---

## 8. 变更管理

### 8.1 源表变更处理

当源系统表结构发生变更时，ODS 层需同步调整：

| 变更类型 | 处理方式 | 说明 |
|---------|---------|------|
| 新增字段 | ODS 表同步新增字段 | 新字段加在最后，历史分区该字段为 NULL |
| 删除字段 | ODS 表保留字段，标记废弃 | 添加 COMMENT 标注 `[DEPRECATED]` |
| 字段类型变更 | 评估兼容性后调整 | 需确认不影响下游 DWD 层 |
| 表名变更 | 新建 ODS 表，旧表保留过渡期 | 过渡期结束后下线旧表 |

### 8.2 变更流程

1. 收到源系统变更通知（或主动监测到变更）
2. 评估变更影响范围（影响哪些 DWD/DWM 表）
3. 修改 ODS 表 DDL 并在测试环境验证
4. 通知下游所有依赖方
5. 在生产环境执行变更
6. 验证数据接入正确性
7. 更新相关文档

---

## 9. 监控告警规范

### 9.1 必配监控项

| 监控项 | 说明 | 告警级别 |
|--------|------|---------|
| 同步任务状态 | 任务执行成功/失败 | P0 — 失败立即告警 |
| 同步完成时间 | 数据产出是否满足 SLA | P0 — 超时告警 |
| 接入数据量 | 数据行数异常（为0或波动过大） | P1 — 波动 > 50% 告警 |
| 源库连接状态 | 无法连接源数据库 | P0 — 连接失败告警 |
| 分区产出 | 分区是否正常产出 | P1 — 分区缺失告警 |

### 9.2 SLA 管理

| 数据类型 | SLA 要求 | 说明 |
|---------|---------|------|
| 核心业务数据 | T+1 凌晨 03:00 前完成 | 交易、支付、用户等核心表 |
| 日志/埋点数据 | T+1 凌晨 05:00 前完成 | APP 日志、行为埋点数据 |
| 外部数据 | T+1 上午 08:00 前完成 | 第三方数据，受对方时效影响 |
| 实时数据 | 延迟 < 5 分钟 | Binlog / Kafka 实时同步链路 |

---

## 10. Checklist

ODS 数据接入开发完成后，请对照以下 Checklist 进行自检：

- [ ] 表名/库名符合 ODS 命名规范
- [ ] DDL 包含完整的 COMMENT（含源字段映射说明）
- [ ] DDL 头部注明数据来源、同步方式、负责人
- [ ] 源表所有字段完整接入，未做裁剪
- [ ] ODS 元数据字段（`ods_create_time`, `ods_source`）已添加
- [ ] 分区策略合理（日期分区，大数据量加小时分区）
- [ ] 增量同步的增量字段选择合理且可靠
- [ ] 接入行数校验已配置
- [ ] 源表行数对比校验已配置
- [ ] 主键唯一性校验已配置（全量表）
- [ ] 核心字段空值检查已配置
- [ ] 同步 SLA 满足下游需求
- [ ] 数据保留策略已配置
- [ ] 变更通知机制已建立
- [ ] 文档已更新（接入说明、字段映射、血缘关系）
