# Kotis 平台 API 契约与数据模型速查

> 配套文档：`Kotis系统拆解与仿制规格书.html`
> 来源：Kotis 官方 API 文档（2026-09-11 更新）+ 官方帮助中心，实测 HTTP 指纹
> 用途：仿制系统对外接口设计的直接参考

---

## 0. 通用约定

| 项 | 约定 |
|---|---|
| 规范 | JSON:API 1.0（`data` / `type` / `attributes` / `relationships` / `included` / `jsonapi.version`） |
| 认证 | `Authorization: <API_KEY>` —— **无 Bearer 前缀，直接放裸 key** |
| Content-Type | `application/json`（写操作） |
| 订单/库存域 | `https://kwf.kotisdesign.com/api/v1` |
| 兑换码/门户域 | `https://stores.kotisdesign.com/api/v1` |
| 错误格式 | `{"errors":[{"title":"...","detail":"...","status":"400"}]}` |

### API Key 生命周期

```
创建（联系客户经理获取链接）
 ├─ 不勾 Live → 测试 Key ──[升为正式]──▶ 正式 Key
 └─ 勾 Live   ─────────────────────────▶ 正式 Key  ← 不可降级回测试
```

- `Notes` 字段：标记用途，便于清理废弃 Key
- **兑换码相关端点必须用正式 Key**
- 正式 Key 创建后不可逆 —— 仿制系统建议同样设计，但可额外提供「测试 Key 独立环境」

---

## 1. 查库存

```http
GET https://kwf.kotisdesign.com/api/v1/skus/quantity
Authorization: <API_KEY>
```

```json
{
  "data": [
    { "id": "33315", "type": "skus",
      "attributes": {
        "physical_quantity": 0,
        "available_quantity": 0,
        "sku": "107529-1-L"
      } },
    { "id": "33316", "type": "skus",
      "attributes": {
        "physical_quantity": 0,
        "available_quantity": 0,
        "sku": "107529-1-M"
      } },
    { "id": "33317", "type": "skus",
      "attributes": {
        "physical_quantity": 0,
        "available_quantity": 0,
        "sku": "107529-1-S"
      } }
  ],
  "jsonapi": { "version": "1.0" }
}
```

**要点**
- `physical_quantity` = 货架实物数
- `available_quantity` = 可下单数（= physical − 已下单未发货）
- **两者必须分离存储**，否则并发下单必然超卖
- SKU 格式 `107529-1-L`：`{款号}-{变体号}-{尺码}`，尺码独立成 SKU

---

## 2. 创建订单

```http
POST https://kwf.kotisdesign.com/api/v1/orders/create
Authorization: <API_KEY>
Content-Type: application/json
```

```json
{
  "email": "test@example.com",
  "shipping_method": "Economy",
  "order_id": "",
  "skus": [
    { "quantity": "3", "sku": "107529-1-S" },
    { "quantity": "1", "sku": "107529-4-OSFA" }
  ],
  "address": {
    "name": "Test Person",
    "attention": "",
    "address1": "123 First Street",
    "address2": "",
    "city": "Seattle",
    "state": "WA",
    "zip": "98102",
    "country": "",
    "phone": "",
    "address_type": ""
  }
}
```

**成功**
```json
{ "status": "success", "order_num": "5134691" }
```

**失败**
```json
{
  "errors": [
    { "title": "Invalid Order", "detail": "Sku 1 does not exist, Sku 2 does not exist" }
  ]
}
```

**要点**
- `quantity` 传**字符串**（`"3"`），不是数字 —— 仿制时应做兼容处理
- `order_id` 留空 = 由 Kotis 分配；传入 = 外部系统自带单号（幂等键）
- `shipping_method`：`Economy` | `FedEx Ground` | `FedEx Express Saver` | `FedEx 2 Day` | `FedEx Standard Overnight` | `UPS Ground` | `UPS 3 Day Select` | `UPS 2nd Day Air` | `UPS Next Day Air`
- `address_type` 用于区分营业地址 / 收货地址
- 国际单需在 `attention` / `phone` 填收件人联系方式（否则卡关务）

---

## 3. 查订单状态与物流

```http
GET https://kwf.kotisdesign.com/api/v1/orders/status?order_numbers=12345,23719201
Authorization: <API_KEY>
```

```json
{
  "data": [
    { "id": "12345", "type": "orders",
      "attributes": { "status": "shipped" },
      "relationships": {
        "shipments": { "data": [
          { "type": "shipments", "id": "543416" },
          { "type": "shipments", "id": "544568" }
        ] }
      } },
    { "id": "23719201", "type": "orders",
      "relationships": {
        "shipments": { "data": [
          { "type": "shipments", "id": "767586" }
        ] }
      } }
  ],
  "included": [
    { "id": "543416", "type": "shipments",
      "attributes": {
        "tracking_numbers": "1Z8424330419842539",
        "shipment_date": "2018-10-02T14:50:24-07:00",
        "delivery_status": "delivered"
      } },
    { "id": "544568", "type": "shipments",
      "attributes": {
        "tracking_numbers": "1Z8424330402599594",
        "shipment_date": "2018-10-05T15:18:28-07:00",
        "delivery_status": "out_for_delivery"
      } },
    { "id": "767586", "type": "shipments",
      "attributes": {
        "tracking_numbers": "9405537897842130146579",
        "shipment_date": "2019-11-12T07:48:06-08:00",
        "status": "in_transit"
      } }
  ],
  "jsonapi": { "version": "1.0" }
}
```

**无效订单号** → `{"data": [], "jsonapi": {"version": "1.0"}}`
**无效 Key** → `{"errors": [{"title": "Invalid Api Key", "detail": ""}]}`

### 状态枚举

| 字段 | 枚举值 |
|---|---|
| `attributes.status`（订单） | `pending` · `partially_shipped` · `shipped` · `delivered` · `cancelled` |
| `delivery_status`（运单） | `unknown` · `pre_transit` · `in_transit` · `out_for_delivery` · `delivered` · `available_for_pickup` · `return_to_sender` · `failure` · `cancelled` · `error` |

**要点**
- 一个订单可对应**多个 shipment**（分批发运）→ 必须建模 `Order 1:N Shipment`
- `included` 侧载同时返回运单，避免 N+1 请求 —— 这是 JSON:API 的核心价值

---

## 4. 按来源拉订单（对账用）

```http
GET https://kwf.kotisdesign.com/api/v1/orders/by_source
Authorization: <API_KEY>

source       = all | portal | api | csv | inventory-management   (必填)
start_date   = 2025-08-01                                       (必填, ISO 8601)
end_date     = 2025-08-31                                       (必填, ISO 8601)
portal_name  = testportalurl                                    (可选, 仅 source=portal 时生效)
```

```json
{ "data": [
    { "order_number": "123456",
      "order_date": "2025-08-02T12:30Z",
      "status": "shipped",
      "source": "portal",
      "portal_name": "testportalurl" } ] }
```

**错误条件**
- `source` 缺失或非法 → `Invalid source`
- `start_date` / `end_date` 缺失或非合法 ISO 8601
- `end_date` < `start_date`
- **`end_date - start_date` > 60 天**

**要点**
- `source` 分类是履约 SLA 与财务对账的关键维度，**必须建字段并全链路保留**
- 60 天窗口限制同时保护后端并引导增量同步 —— 值得照抄

---

## 5. 批量生成兑换码

```http
POST https://stores.kotisdesign.com/api/v1/redemption_codes/create
Authorization: <API_KEY>
```

### 必填

| 字段 | 说明 |
|---|---|
| `portal_slug` | 门户 slug（URL 名） |
| `number_to_create` | 生成数量，接受数字或数字字符串 |

### 可选

| 字段 | 说明 |
|---|---|
| `points` | 每码积分额度。**用积分的门户必填；不用积分的门户传了会报错；必须 > 0** |
| `email` | 绑定邮箱并立即发送。**仅 `number_to_create = 1` 时可用** |
| `first_name` | **依赖 `email`** |
| `last_name` | **依赖 `email`** |

### 限额

- 需**正式 Key**
- 日生成上限 **5,000** 个
- 年生成上限 **100,000** 个
- 接近上限且请求数超剩余额度 → 返回校验消息说明最多可生成多少

### 示例

```json
// 批量普通码
{ "portal_slug": "YOUR_PORTAL_SLUG", "number_to_create": 100 }

// 批量积分码
{ "portal_slug": "YOUR_PORTAL_SLUG", "number_to_create": 100, "points": 100 }

// 单个带名发邮件
{ "portal_slug": "YOUR_PORTAL_SLUG", "number_to_create": 1,
  "email": "recipient@example.com", "first_name": "Jane", "last_name": "Doe" }
```

---

## 6. 查兑换码状态

```http
GET https://stores.kotisdesign.com/api/v1/redemption_codes/status
Authorization: <API_KEY>

portal_slug = "YOUR_PORTAL_SLUG"
code        = "CODE_TO_LOOKUP"
```

```json
{
  "data": {
    "code": "CODE_TO_LOOKUP",
    "status": "unclaimed",
    "opened": true,
    "opened_date": "2020-12-02T14:55:41.750-08:00",
    "claimed_date": null,
    "order_number": null
  },
  "status": "ok"
}
```

| `status` | 含义 |
|---|---|
| `unclaimed` | 未核销 |
| `claimed` | 已核销 |
| `hidden` | 已作废 |

**错误** → HTTP 400 / 404，`{"errors":[{"title":"Code not found","detail":""}]}`

**要点**
- `opened` / `opened_date` = 用户打开了兑换页但未下单 —— 这是「催促未核销」功能的数据基础
- `order_number` 与订单强关联 → 兑换码同时是预算凭据和发放台账

---

## 7. 数据模型（PostgreSQL 草案）

```sql
-- ========== 租户与门户 ==========
CREATE TABLE clients (
  id            BIGSERIAL PRIMARY KEY,
  name          TEXT NOT NULL,
  slug          TEXT UNIQUE NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE programs (
  id            BIGSERIAL PRIMARY KEY,
  client_id     BIGINT NOT NULL REFERENCES clients(id),
  name          TEXT NOT NULL,
  -- 配额
  sku_quota     INTEGER,
  pallet_quota  INTEGER,
  -- 平台费豁免
  platform_fee_cents      INTEGER DEFAULT 150000,   -- $1,500/季
  spend_threshold_cents   INTEGER DEFAULT 500000,   -- $5,000/季
  UNIQUE(client_id, name)
);

CREATE TABLE portals (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  slug           TEXT UNIQUE NOT NULL,
  title          TEXT NOT NULL,
  portal_type    TEXT NOT NULL
                 CHECK (portal_type IN ('non_credit_card','credit_card','redemption','pre_order')),
  -- 品牌配置
  brand_config   JSONB DEFAULT '{}'::jsonb,   -- colors/fonts/header/banner
  -- 交易与管控开关
  settings       JSONB DEFAULT '{}'::jsonb,   -- show_price/free_shipping/
                                                -- need_by_enabled/approval_required/
                                                -- access_code/budget_enabled/...
  CHECK (portal_type = 'credit_card' OR portal_type = 'pre_order'
         OR NOT (settings ? 'payment'))
);

-- ========== 商品与库存 ==========
-- 款式（Style）与变体（Variant）两层，避免 SKU 爆炸
CREATE TABLE styles (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  name           TEXT NOT NULL,
  category       TEXT,
  color          TEXT,
  description    TEXT,
  value_each_cents INTEGER NOT NULL DEFAULT 0,
  hazmat         BOOLEAN DEFAULT false,
  customer_item_no TEXT,
  access_group_id BIGINT,
  active         BOOLEAN DEFAULT true
);

CREATE TABLE variants (                -- 一个变体 = 一个 SKU
  id             BIGSERIAL PRIMARY KEY,
  style_id       BIGINT NOT NULL REFERENCES styles(id),
  sku            TEXT UNIQUE NOT NULL,   -- 例 107529-1-L
  size           TEXT,
  barcode        TEXT
);

CREATE TABLE inventory (
  variant_id     BIGINT PRIMARY KEY REFERENCES variants(id),
  physical_qty   INTEGER NOT NULL DEFAULT 0 CHECK (physical_qty >= 0),
  available_qty  INTEGER NOT NULL DEFAULT 0 CHECK (available_qty >= 0),
  reserved_qty   INTEGER NOT NULL DEFAULT 0,
  low_stock_threshold INTEGER DEFAULT 0,
  version        INTEGER NOT NULL DEFAULT 0   -- 乐观锁
);

CREATE TABLE inventory_adjustments (
  id             BIGSERIAL PRIMARY KEY,
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  adjustment     INTEGER NOT NULL,          -- +/- 
  reason         TEXT NOT NULL,              -- 'Receiving Error' 等
  counted_never_existed BOOLEAN DEFAULT false, -- 不计入 2% 阈值
  operator_id    BIGINT,
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE incoming_inventory_requests (   -- IIR
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  reference_no   TEXT NOT NULL,
  est_arrival    DATE,
  vendor         TEXT,
  tracking       TEXT,
  notes          TEXT,
  status         TEXT NOT NULL DEFAULT 'in_transit',
  delivered_at   DATE,
  received_at    TIMESTAMPTZ
);

CREATE TABLE iir_lines (
  id             BIGSERIAL PRIMARY KEY,
  iir_id         BIGINT NOT NULL REFERENCES incoming_inventory_requests(id),
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  qty            INTEGER NOT NULL
);

CREATE TABLE kits (                        -- 预组套，计 1 个 SKU
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  name           TEXT NOT NULL,
  variant_id     BIGINT REFERENCES variants(id)  -- 组套本身作为可下单变体
);

CREATE TABLE kit_components (
  kit_id         BIGINT NOT NULL REFERENCES kits(id),
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  qty            INTEGER NOT NULL DEFAULT 1
);

-- ========== 上架（Portal × SKU）============
CREATE TABLE listings (
  id             BIGSERIAL PRIMARY KEY,
  portal_id      BIGINT NOT NULL REFERENCES portals(id),
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  name           TEXT,
  description    TEXT,
  price_cents    INTEGER,
  activated      BOOLEAN DEFAULT true,
  -- Advanced 配置项
  highlight_text TEXT,
  custom_input_label TEXT,
  custom_dropdown_values TEXT[],
  deactivate_at  TIMESTAMPTZ,
  hold_orders_notify TEXT,
  size_chart_url TEXT,
  free_shipping  BOOLEAN DEFAULT false,
  ready_to_activate BOOLEAN DEFAULT false,
  suggested      BOOLEAN DEFAULT false,
  backorderable  BOOLEAN DEFAULT false,
  delivery_notes TEXT,
  UNIQUE(portal_id, variant_id)
);

-- ========== 用户与权限 ==========
CREATE TABLE users (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  email          TEXT NOT NULL,
  password_hash  TEXT,
  role           TEXT NOT NULL DEFAULT 'member'
                 CHECK (role IN ('owner','manager','accounting','member')),
  sso_provider   TEXT,
  sso_subject    TEXT,
  UNIQUE(program_id, email)
);

CREATE TABLE access_groups (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  name           TEXT NOT NULL
);

CREATE TABLE user_access_groups (
  user_id        BIGINT NOT NULL REFERENCES users(id),
  access_group_id BIGINT NOT NULL REFERENCES access_groups(id),
  PRIMARY KEY (user_id, access_group_id)
);

-- 预算组：承载预算 + 审批人 + 结算页下拉 + 报表分组
CREATE TABLE budget_groups (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  name           TEXT NOT NULL,
  approver_email TEXT,
  budget_cents   BIGINT DEFAULT 0,
  spent_cents    BIGINT DEFAULT 0,
  period_start   DATE,
  period_end     DATE,
  custom_option_label TEXT,
  custom_option_values TEXT[]
);

CREATE TABLE budget_group_members (
  budget_group_id BIGINT NOT NULL REFERENCES budget_groups(id),
  user_id        BIGINT NOT NULL REFERENCES users(id),
  PRIMARY KEY (budget_group_id, user_id)
);

-- ========== 订单 ==========
CREATE TABLE orders (
  id             BIGSERIAL PRIMARY KEY,
  order_number   TEXT UNIQUE NOT NULL,
  program_id     BIGINT NOT NULL REFERENCES programs(id),
  portal_id      BIGINT REFERENCES portals(id),
  source         TEXT NOT NULL
                 CHECK (source IN ('portal','api','csv','inventory-management')),
  status         TEXT NOT NULL DEFAULT 'pending',
  email          TEXT NOT NULL,
  -- 金额
  subtotal_cents     BIGINT DEFAULT 0,
  shipping_cents     BIGINT DEFAULT 0,
  discount_cents     BIGINT DEFAULT 0,
  tax_cents          BIGINT DEFAULT 0,
  tax_rate           NUMERIC(6,4),
  total_cents        BIGINT DEFAULT 0,
  gift_card_cents    BIGINT DEFAULT 0,
  gift_card_codes    TEXT[],
  non_credit_subtotal_cents BIGINT DEFAULT 0,
  -- 履约
  requested_ship_method TEXT,
  need_by_date      DATE,
  ship_on_or_after  DATE,
  instructions      TEXT,
  short_ship        BOOLEAN DEFAULT false,
  po_number         TEXT,
  additional_data_label TEXT,
  additional_data   TEXT,
  -- 管控
  approver_email    TEXT,
  budget_group_id   BIGINT REFERENCES budget_groups(id),
  location          TEXT,
  custom_option_label TEXT,
  custom_option     TEXT,
  -- 促销
  discount_code     TEXT,
  promotion_code    TEXT,
  gift_notes        TEXT,
  gift_notes_handwritten BOOLEAN DEFAULT false,
  marketing_optin   BOOLEAN DEFAULT false,
  -- 幂等与外部引用
  api_external_order_no TEXT,
  csv_unique_key    TEXT,
  created_at        TIMESTAMPTZ DEFAULT now(),
  approved_at       TIMESTAMPTZ
);

CREATE TABLE order_lines (
  id             BIGSERIAL PRIMARY KEY,
  order_id       BIGINT NOT NULL REFERENCES orders(id),
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  order_qty      INTEGER NOT NULL,
  original_order_qty INTEGER NOT NULL,   -- 保留原始量，支撑中途取消
  price_cents    BIGINT,
  value_each_cents INTEGER
);

CREATE TABLE approvals (
  id             BIGSERIAL PRIMARY KEY,
  order_id       BIGINT NOT NULL REFERENCES orders(id),
  approver_email TEXT,
  status         TEXT NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','approved','rejected')),
  shipping_method_override TEXT,
  need_by_date_override DATE,
  qty_overrides  JSONB,          -- {order_line_id: new_qty}
  reject_reason  TEXT,
  created_at     TIMESTAMPTZ DEFAULT now(),
  decided_at     TIMESTAMPTZ
);

-- ========== 发货 ==========
CREATE TABLE shipments (
  id             BIGSERIAL PRIMARY KEY,
  order_id       BIGINT NOT NULL REFERENCES orders(id),
  ship_method    TEXT,
  tracking_numbers TEXT,
  shipment_date  TIMESTAMPTZ,
  delivery_status TEXT,
  boxes          INTEGER,
  subtotal_cents BIGINT DEFAULT 0,
  shipping_cents BIGINT DEFAULT 0,
  discount_cents BIGINT DEFAULT 0,
  tax_cents      BIGINT DEFAULT 0,
  total_cents    BIGINT DEFAULT 0,
  gift_card_cents BIGINT DEFAULT 0
);

CREATE TABLE shipment_lines (
  id             BIGSERIAL PRIMARY KEY,
  shipment_id    BIGINT NOT NULL REFERENCES shipments(id),
  order_line_id  BIGINT REFERENCES order_lines(id),
  variant_id     BIGINT NOT NULL REFERENCES variants(id),
  shipped_qty    INTEGER NOT NULL,
  return_label_included BOOLEAN DEFAULT false
);

CREATE TABLE shipping_charges (
  id             BIGSERIAL PRIMARY KEY,
  order_id       BIGINT REFERENCES orders(id),
  charge_date    DATE,
  ship_method    TEXT,
  amount_cents   BIGINT NOT NULL,
  kind           TEXT   -- 'shipping' | 'duty' | 'tax' | 'address_correction' ...
);

-- ========== 兑换码 / 礼品卡 / 折扣码 ==========
CREATE TABLE redemption_codes (
  id             BIGSERIAL PRIMARY KEY,
  portal_id      BIGINT NOT NULL REFERENCES portals(id),
  code           TEXT UNIQUE NOT NULL,
  points         INTEGER,
  status         TEXT NOT NULL DEFAULT 'unclaimed'
                 CHECK (status IN ('unclaimed','claimed','hidden')),
  opened         BOOLEAN DEFAULT false,
  opened_date    TIMESTAMPTZ,
  claimed_date   TIMESTAMPTZ,
  order_id       BIGINT REFERENCES orders(id),
  email          TEXT,
  first_name     TEXT,
  last_name      TEXT,
  created_at     TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE gift_cards (
  id             BIGSERIAL PRIMARY KEY,
  portal_id      BIGINT NOT NULL REFERENCES portals(id),
  code           TEXT UNIQUE NOT NULL,
  amount_cents   BIGINT NOT NULL,
  status         TEXT NOT NULL DEFAULT 'active'
                 CHECK (status IN ('active','used','hidden')),
  used_at        TIMESTAMPTZ
);

CREATE TABLE discount_codes (
  id             BIGSERIAL PRIMARY KEY,
  portal_id      BIGINT NOT NULL REFERENCES portals(id),
  code           TEXT UNIQUE NOT NULL,
  kind           TEXT CHECK (kind IN ('percent','fixed')),
  value          NUMERIC(12,2),
  max_uses       INTEGER,
  used_count     INTEGER DEFAULT 0,
  status         TEXT DEFAULT 'active',
  expires_at     TIMESTAMPTZ
);

-- ========== 退货 ==========
CREATE TABLE return_authorizations (
  id             BIGSERIAL PRIMARY KEY,
  ra_number      TEXT UNIQUE NOT NULL,
  order_id       BIGINT NOT NULL REFERENCES orders(id),
  return_type    TEXT CHECK (return_type IN ('refund','exchange')),
  requested_at   TIMESTAMPTZ DEFAULT now(),
  requested_by   TEXT,
  notes          TEXT,
  exchange_details TEXT,
  refunded_cents BIGINT DEFAULT 0
);

CREATE TABLE return_lines (
  id             BIGSERIAL PRIMARY KEY,
  ra_id          BIGINT NOT NULL REFERENCES return_authorizations(id),
  order_line_id  BIGINT REFERENCES order_lines(id),
  qty            INTEGER NOT NULL,
  price_cents    BIGINT
);

-- ========== 审计 ==========
CREATE TABLE audit_logs (
  id             BIGSERIAL PRIMARY KEY,
  program_id     BIGINT,
  actor_id       BIGINT,
  action         TEXT NOT NULL,
  entity_type    TEXT NOT NULL,
  entity_id      TEXT,
  before_json    JSONB,
  after_json     JSONB,
  ip             INET,
  created_at     TIMESTAMPTZ DEFAULT now()
);
```

---

## 8. PII 脱敏字段清单

导出报表时必须打码（保留 City / State / Zip / Country）：

| 打码 | 保留 |
|---|---|
| Email | Order# |
| Ship-To | City |
| Attn | State |
| Address1 | Zip |
| Address2 | Country |

原因字段：`Order Date`、`Order Subtotal`、`Portal`、`SKU` 等不脱敏。

---

## 9. 实现要点清单

1. **库存双量分离** —— `physical_qty` / `available_qty` / `reserved_qty` 三字段，加 `version` 乐观锁
2. **订单态与发货态分离** —— 季度账单 Portal Sales 取 `shipments.subtotal_cents`，**不取** `orders.subtotal_cents`
3. **`original_order_qty` 必留** —— 支撑「客户中途取消 1 件」后的数量与金额对账
4. **`source` 字段全链路保留** —— portal / api / csv / inventory-management
5. **预算扣减与审批动作同事务** —— 避免竞态；订单加版本号
6. **兑换码单次使用 + 积分上限** —— `claimed` 后剩余积分作废；`opened` 与 `claimed` 分离支持催核销
7. **审批人的修改必须留痕** —— 存 `qty_overrides` / `shipping_method_override` / `need_by_date_override`，不静默覆盖
8. **PII 在序列化层统一脱敏** —— 不在各个报表里各写一遍
9. **多租户强制隔离** —— 所有查询带 `program_id`；建议加 PostgreSQL RLS 或 ORM 全局 scope
10. **API 60 天查询窗口 + 兑换码日/年限额** —— 保护后端并引导增量同步
