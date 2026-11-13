# Cyndi Global 对标 Kotis Design：平台能力缺口与搭建规划

> **归档说明（2026-11-13）**
>
> 本文件是同日 Word 交付稿《Cyndi对标Kotis平台能力缺口与搭建规划.docx》的 Markdown 同源归档，便于在 GitHub 上直接修订。
> 内容聚焦 **Kotis 公开 API Documentation 页 + Integrations 分类**，与仓库既有资料的分工：
>
> | 需要什么 | 看哪份 |
> |---|---|
> | Kotis 全平台能力基线（WMS、报表、SLA、计费） | `Kotis系统拆解与仿制规格书.html` |
> | 对外接口字段与建表 DDL | `API契约与数据模型速查.md` |
> | 2026-10-29 现状差距矩阵 | `落地差距分析.md` |
> | **两套资料的对照、现状校正与待决策分歧** | **`整合汇总与修订台账.md`（修订先读）** |
>
> ⚠️ 本文中的优先级、路线图与「不建议照搬」结论与 10-29 规格存在分歧，**以 `整合汇总与修订台账.md` §4 的决策栏为准**；
> 本文保留原稿表述不作改写，便于对照修正。原文中的「八个接口」指页面章节数，其中带端点的为 6 个，详见 `API契约与数据模型速查.md`。
>
> 修订本文件后，请按台账 §8 同步清单检查需要一起改的文件。

---


**文档性质**：竞品能力分析报告与平台建设方案
**分析对象**：Kotis Design 公开支持文档《API Documentation》及其 Integrations 知识库体系
**编制日期**：2026 年 11 月 13 日
**适用范围**：Cyndi Global 选品展示站、订单系统、库存系统与后台管理平台的后续建设

---

## 一、执行摘要

### 1.1 一句话结论

竞品这一页并不是一个"网页设计参考"，而是一份**面向企业客户的系统集成说明书**。它说明 Kotis Design 售卖的不是单件定制服装，而是一套可以被客户 HR 系统、CRM、电商平台和内部采购流程调用的「在线礼品计划平台」。Cyndi Global 目前完成的是第一层能力（站内选品展示、下单、库存管理），缺口集中在第二层（对外 API、履约可视化、对账）与第三层（客户身份、自助兑换、生态集成）。

### 1.2 三个必须先补的地基

1. **统一 SKU 主数据**：竞品所有库存、下单、状态查询都以 `107529-1-S` 这类字符串 SKU 为唯一键；Cyndi 目前以 `productId + colorId + size` 组合标识，且部分颜色缺少稳定 id，尚不具备对外可引用的唯一编码。
2. **对外开放 API 与密钥治理**：竞品提供 API Key（含 test / live 双环境）、三个交易端点和统一错误格式；Cyndi 的现有接口全部依赖浏览器会话或后台会话，客户系统无法调用。
3. **订单状态机与多批次物流追踪**：竞品把订单状态拆成 5 档、物流状态拆成 10 档，并允许一个订单关联多个 shipment；Cyndi 目前缺少「部分发货」与「按批次追踪」的表达能力。

### 1.3 建议的板块清单与优先级

| 优先级 | 板块 | 解决的核心问题 | 主要依赖 |
|---|---|---|---|
| P0 | A 统一 SKU 主数据与可售库存口径 | 对外没有稳定商品标识 | 现有 `inventory`、`colors_json` |
| P0 | B 对外开放 API 与 API Key 治理 | 客户系统无法自动下单 | A、服务端定价逻辑 |
| P0 | C 订单状态机与多批次物流追踪 | 大客户无法自助查货 | 现有 `shipments` |
| P1 | D 订单来源与对账报表 | 渠道、履约、财务无法量化 | `orders` 字段改造 |
| P1 | E 客户自助兑换与额度（Portal 2.0） | 依赖人工收集 Excel 与地址 | A、B、gate 登录 |
| P1 | F 企业客户身份与权限 | 单一共享账号，无法分权分预算 | gate 会话体系 |
| P2 | G 生态集成与批量导入 | 无法嵌入客户既有系统 | B、D |
| P2 | H 对外文档中心 | 集成能力无法被销售与交付复用 | B 定稿 |

> 判断依据：凡是需要客户 IT 系统参与的能力，都必须先有稳定 SKU 编码与对外接口，否则后续每一层都会返工。

---

## 二、竞品页面解读：这页到底是什么

### 2.1 页面身份

| 项目 | 内容 |
|---|---|
| 页面标题 | API Documentation |
| 所属站点 | Kotis Design Support（基于 Forumbee 的公开知识库） |
| 所属分类 | Integrations，上级为 My Online Program |
| 页面地址 | `https://kotisdesign.forumbee.com/t/83hvjff/api-documentation` |
| 页面更新时间 | 2026 年 9 月 12 日（页面标注 "updated 2 mths ago"） |
| 可见性 | 公开可读，无需登录 |
| 文档性质 | B2B 集成接口说明，不是营销页、不是产品目录 |

页面本身十分克制：没有截图、没有设计装饰，只有章节、请求示例、响应示例、字段表和错误枚举。但它透露出竞品的商业定位——**礼品计划的后台能力是可以被客户系统直接调用的**。

### 2.2 文档结构与八个接口

| 序号 | 章节 | 内容要点 | 透露的能力信号 |
|---|---|---|---|
| 1 | Getting started with the API | 创建 API Key 的链接需联系 Kotis 客户经理；提示密钥应安全保存、最小范围分发 | 集成是"客户经理开通"的销售动作，不是自助注册 |
| 2 | Creating an API key | `Live` 勾选框生成正式密钥；未勾选为测试密钥，不影响生产；测试密钥可转正式但**不可逆转**；`Notes` 字段用于记录用途、便于清理失效密钥 | 有沙盒/正式环境隔离与密钥生命周期管理 |
| 3 | Check inventory | `GET https://kwf.kotisdesign.com/api/v1/skus/quantity`，返回 `physical_quantity`、`available_quantity`、`sku` | 区分实物库存与可售库存；SKU 是唯一键 |
| 4 | Create an order | `POST /api/v1/orders/create`，入参含 `email`、`shipping_method`、`order_id`（客户方单号）、`skus[{quantity, sku}]`、`address` | 客户可在自己的系统里直接下单，并携带自己的单号用于对账 |
| 5 | Check order status | `GET /api/v1/orders/status?order_numbers=...`，JSON:API 结构，订单通过 `relationships.shipments` 关联多个发货批次，`included` 中给出追踪号与配送状态 | 一单多批次、追踪号挂在 shipment 层、批量查询 |
| 6 | Get orders by source | `GET /api/v1/orders/by_source`，`source` 取值为 `all / portal / api / csv / inventory-management`，ISO 8601 起止日期，跨度不得超过 60 天 | 订单来源被显式建模，可对账、可分渠道分析 |
| 7 | Create redemption codes | `POST https://stores.kotisdesign.com/api/v1/redemption_codes/create`，`portal_slug`、`number_to_create`、可选 `points / email / first_name / last_name`；日限 5,000、年限 100,000 | 面向员工礼品场景的兑换码与积分体系，可批量生成并邮件触达 |
| 8 | Get redemption code status | `GET /api/v1/redemption_codes/status`，返回 `claimed / unclaimed / hidden`、打开时间、认领时间、关联订单号 | 兑换码全生命周期可追踪，直接回连订单 |

### 2.3 可直接借鉴的规范与枚举

竞品把「状态」写成了机器可读的枚举，这是最容易被忽略、却最能减少交付扯皮的部分。

| 类别 | 枚举值 |
|---|---|
| 订单状态 | `pending`、`partially_shipped`、`shipped`、`delivered`、`cancelled` |
| 配送状态 | `unknown`、`pre_transit`、`in_transit`、`out_for_delivery`、`delivered`、`available_for_pickup`、`return_to_sender`、`failure`、`cancelled`、`error` |
| 兑换码状态 | `claimed`、`unclaimed`、`hidden` |
| 订单来源 | `all`、`portal`、`api`、`csv`、`inventory-management` |
| 错误响应 | `{"errors":[{"title":"Invalid Order","detail":"Sku 1 does not exist, Sku 2 does not exist"}]}` |

其中两点值得特别注意：

- **一次性返回全部校验错误**：建单失败时不是只报第一个错误，而是把不存在的 SKU 全部列出，便于客户系统一次性修正。
- **JSON:API 风格**：用 `data / attributes / relationships / included` 表达主资源与关联资源，避免把所有字段平铺成一个大对象。

### 2.4 页面之外：竞品的完整集成版图

该页所属的 Integrations 分类下还有五篇文档，说明 API 只是整个集成体系的一环：

| 文档 | 含义 |
|---|---|
| Workday x Kotis - Scheduling Work Anniversaries and New Hires | 用 Workday 自定义报表提前排期员工周年礼、新人礼 |
| Single Sign-On | 用客户公司身份提供商登录礼品计划 |
| Enabling multi-factor authentication | 多因素认证 |
| Kotis Salesforce App - Install Guide | Salesforce AppExchange 应用 |
| Shopify Install Guide | 与 Shopify 店铺集成 |

知识库还按客户旅程划分了 My Online Program、Users, user groups & budgeting、Admin access、Website customization、Analytics & reporting、Warehouse & Shipping、Track my order、Quarterly billing、Ordering Tips 等分类。

**核心洞察**：Kotis 的壁垒不在商品页面，而在于「把企业客户的人员、预算、审批、收货、对账全部接进自己的平台」。API 文档只是这堵墙露出水面的那一截。Cyndi 若只对比商品展示与价格，会低估差距；若从「客户采购流程自动化」角度看，差距是清晰且可分批补齐的。

---

## 三、Cyndi Global 现状盘点

以下为本次规划的能力基线，依据项目现有系统记录整理。

| 领域 | 已具备 | 说明 |
|---|---|---|
| 前端展示 | 选品站 `/showcase/`，9 个分类约 44 款，桌面与手机菜单、gate 登录、色板图、Size Chart 多图、图库灯箱 | 展示层完整度较高，视觉与移动端已完成专项优化 |
| 主站与后台 | `/`、`/shop`、`/admin/`、`/admin/showcase.html`、`/admin/inventory`、`/admin/orders.html` | 后台可编辑产品、价格、图片、库存、订单状态 |
| 数据存储 | Cloudflare D1：`showcase_*`、`orders`、`inventory`、`order_stock_lines`、`inventory_movements` 等；图片在 R2 | 服务端权威数据，不再依赖浏览器 localStorage |
| 订单 | 服务端生成订单号 `SHOWCASE-YYYYMMDD-NNN`；服务端定价、MOQ 与阶梯价校验；改价后自动重算 PENDING 订单；金额全站千分位 | 已有真实客户订单：14 行、5,500 件、`$85,451.00` |
| 库存 | SKU 行视图（一个货号加一个颜色为一行）；字段 `available / reserved / incoming / damaged / threshold`；有行即计库存，无行按单生产不限量；库存流水 | 期初已按真实订单重置，共 14 行 / 12 产品 |
| 接口 | `functions/api/showcase/*`（需 showcase 会话）、`functions/api/admin/*`（需后台会话） | 全部面向浏览器，无面向客户系统的机器接口 |
| 标识 | 产品以 `productId + colorId + size` 标识，展示层用 `2027-03` 之类货号 | 尚无全局唯一、可对外引用的 SKU 字符串 |
| 履约 | 已有发货/追踪相关表与状态模块 | 未形成「一单多批次 + 配送状态枚举」的对外表达 |

**基线结论**：Cyndi 已经把「报价 → 选品 → 下单 → 库存」这条内部链路跑通，但所有能力都封闭在自己的页面里。下一步不是继续加商品，而是把这条链路**变成可被外部系统使用的服务**。

---

## 四、能力差距对照表

| 能力项 | Kotis Design | Cyndi Global 现状 | 差距 | 建议动作 |
|---|---|---|---|---|
| 稳定 SKU 编码 | `107529-1-S` 字符串贯穿库存/下单/状态 | 组合键，颜色 id 曾普遍缺失 | 高 | 板块 A |
| 可售库存查询 API | `GET /skus/quantity` | 仅后台页面可见 | 高 | 板块 A + B |
| 客户系统直接下单 | `POST /orders/create` | 仅网页购物车 | 高 | 板块 B |
| 客户方单号与幂等 | `order_id` 字段 | 仅服务端订单号 | 中高 | 板块 B |
| API Key 与测试环境 | test / live 双态、Notes、不可逆 | 无 | 高 | 板块 B |
| 订单状态机 | 5 档，含部分发货 | 状态较简 | 中高 | 板块 C |
| 多批次追踪 | 订单关联多个 shipment，各带追踪号 | 未形成对外结构 | 中高 | 板块 C |
| 订单来源建模 | `portal / api / csv / inventory-management` | 靠订单号前缀区分 | 中 | 板块 D |
| 按来源/日期对账 | `GET /orders/by_source` | 后台列表 | 中 | 板块 D |
| 兑换码与积分 | 批量生成、邮件发送、状态查询、限额 | 无 | 中 | 板块 E |
| SSO / 用户组 / 预算 | 完整支持 | 单一共享账号 | 中 | 板块 F |
| HR / CRM / 电商集成 | Workday、Salesforce、Shopify | 无 | 低（长期） | 板块 G |
| 对外文档中心 | 公开知识库、按客户旅程分类 | 无 | 中 | 板块 H |

---

## 五、需要搭建的板块（详细设计）

### 5.1 板块 A｜统一 SKU 主数据与可售库存口径

**目标**：让每一个「货号 + 颜色 + 尺码」在订单、库存、接口、报表中都指向同一个不可变字符串。

**为什么要做**：竞品所有接口都以 SKU 为唯一键。没有这层，客户系统无法引用商品，API 下单、库存同步、物流回传都会退化成「按名称猜」，且颜色改名或排序调整就会造成错单。Cyndi 库存页虽然已改成 SKU 行视图，但那是展示层的行，不是可对外引用的编码。

**功能清单**

1. 制定编码规则，建议 `货号-色码-尺码`，例如 `2027-03-EGG-L`、`2027-03-DVB-M`；色码取颜色名前两三个字母并做唯一性校验，尺码沿用现有 size 字符串。
2. 在 `inventory` 表新增 `sku_code TEXT`，建立唯一索引；`order_stock_lines` 与 `order_items` 冗余存储该编码，避免联表漂移。
3. 批量回填历史数据：先给缺 id 的颜色补稳定 id 并回写 `colors_json`，再按「有 inventory 行」的记录生成编码；14 行期初数据可一次性处理。
4. 明确两个口径：`physical_quantity`（仓库实物）与 `available_quantity`（可售 = 实物减预留减破损）。对外接口只暴露可售量；无库存行的产品按现有规则返回「按单生产、不限量」的显式标记，而不是返回空或 0。
5. 在后台库存页显示 SKU 编码列，并支持按编码搜索、复制。

**实施注意**：项目约定 D1 加列必须使用 `ALTER TABLE ADD COLUMN`，且 `CREATE TABLE` 与 `MIGRATIONS` 两处同步补齐；回填应通过后端维护接口执行并先备份，当前环境没有 D1 直连 SQL 通道。

**验收标准**：所有有库存行的 SKU 编码唯一且非空；同一 SKU 在订单行、库存流水、接口响应三处完全一致；颜色改名后编码不变。

**优先级 / 依赖**：P0；无前置依赖，是其余板块的地基。

### 5.2 板块 B｜对外开放 API 与 API Key 治理

**目标**：提供一组独立于网页会话的机器接口，让客户采购系统可以查库存、下单、查状态。

**为什么要做**：这是竞品页面的主体。大客户往往要求「订单从我们的系统发起」，没有 API 就只能靠人工 Excel 往返，既慢又容易错。Cyndi 已有 Excel 下单脚本，但那是内部工具，不是客户可自助使用的能力。

**功能清单**

1. 新建 `functions/api/v1/` 命名空间，与现有 `functions/api/showcase/*`、`functions/api/admin/*` 隔离；认证方式为 `Authorization` 请求头携带 API Key，不复用浏览器 cookie。
2. 新增 `api_keys` 表：`key_hash`、`mode`（`test | live`）、`notes`、`scopes`、`created_at`、`last_used_at`、`revoked_at`；明文密钥只在创建时返回一次。
3. 借鉴 test / live 双环境：`test` 模式的下单请求走完整校验但不写生产订单表，返回模拟订单号；`live` 才真实落单。密钥可吊销、可轮换；Notes 字段强制填写用途。
4. 最小三个端点：

| 方法 | 路径 | 用途 | 关键入参 |
|---|---|---|---|
| GET | `/api/v1/skus/quantity` | 可售库存 | 可选 `sku`、`product_id` |
| POST | `/api/v1/orders/create` | 创建订单 | `external_order_no`、`skus[{sku, quantity}]`、`address`、`email`、`shipping_method` |
| GET | `/api/v1/orders/status` | 订单与追踪 | `order_numbers`（逗号分隔） |

5. **复用服务端定价**：接口只接受 SKU 与数量，单价、总价、创建时间一律由服务端生成；MOQ、颜色、尺码、阶梯价校验沿用现有下单逻辑，绝不信任客户端传价。
6. 幂等：以 `external_order_no + api_key_id` 建唯一约束，重复请求返回首次结果，不产生第二张订单。
7. 统一错误结构：`{"errors":[{"title":"...","detail":"...","status":"400"}]}`，建单校验一次性返回全部错误行。
8. 审计与限流：`api_request_logs` 记录 key、端点、状态码、耗时；按 key 做速率限制；受保护响应统一 `Cache-Control: private, no-store`。

**验收标准**：test key 下单后生产 D1 订单数不变；同一 `external_order_no` 重复提交只生成一张单；错误响应含全部校验失败项；吊销后的 key 立即返回 401。

**优先级 / 依赖**：P0；依赖板块 A 与现有服务端定价模块。

### 5.3 板块 C｜订单状态机与多批次物流追踪

**目标**：让一张订单可以分多批发货，且每批都有独立追踪号与配送状态。

**为什么要做**：定制周边常常分品类、分尺码到货，一次发不完。竞品用 `partially_shipped` 明确表达这一状态，并把追踪号挂在 shipment 而不是订单上。缺少这层时，客户只能看到「已发货」或「未发货」，客服需要反复人工解释。

**功能清单**

1. 订单状态扩展为 `pending / partially_shipped / shipped / delivered / cancelled`，与现有 PENDING 等状态做映射迁移。
2. 建立订单 1:N shipment 关系；`shipments` 增补 `tracking_numbers`、`shipment_date`、`delivery_status`、`carrier`，订单状态由各批次自动汇总（全部发出 → shipped，全部妥投 → delivered）。
3. 配送状态采用竞品 10 档枚举，前端与接口共用同一份常量，避免中英文状态各写一套。
4. 状态变更写 `order_status_history`，便于客服回溯与对账。
5. `/api/v1/orders/status` 返回结构借鉴 JSON:API：主资源为订单，关联 shipment 以数组给出，追踪明细放在 `included` 或同级 `shipments` 数组。

**验收标准**：一单拆两批次发货时，订单显示「部分发货」，两个追踪号分别可查；全部妥投后订单自动变为 delivered；历史 PENDING 订单迁移后状态不丢失。

**优先级 / 依赖**：P0；依赖现有 shipments 与状态模块。

### 5.4 板块 D｜订单来源与对账报表

**目标**：把「这张单从哪来」变成数据字段，并据此做渠道、履约与财务对账。

**为什么要做**：竞品把来源显式建模为 `portal / api / csv / inventory-management`，并提供按来源和日期窗口的查询。Cyndi 目前靠订单号前缀（`SHOWCASE-` 与商城 `SW-`）区分业务线，无法回答「这个客户本月多少个订单来自 API」「哪类订单履约最慢」。

**功能清单**

1. `orders` 新增 `source`（`showcase_portal / admin_manual / api / csv / inventory_management`）、`portal_name`、`external_order_no`；历史订单按前缀与创建方式回填。
2. 新增 `GET /api/v1/orders/by_source`，参数为 `source`、`start_date`、`end_date`（ISO 8601）、可选 `portal_name`；参数校验包含来源合法性、日期格式、起止顺序、跨度上限。
3. 报表视图：按来源/客户/月份的订单数与金额、平均履约时长、库存周转、取消率；金额显示沿用全站千分位规范。
4. 与财务流程衔接：按来源导出对账单，支持与客户方单号勾稽。

**验收标准**：新订单必带 `source`；历史订单回填后可按来源筛选；按日期窗口查询结果与后台订单列表一致。

**优先级 / 依赖**：P1；依赖板块 B（外部单号）。

### 5.5 板块 E｜客户自助兑换与额度（Portal 2.0）

**目标**：把「客户发 Excel、Cyndi 人工整理收货地址」变成「客户员工凭兑换码自助选品下单」。

**为什么要做**：这是竞品页面里最值得借鉴的**商业模式**而非技术细节。兑换码 + 积分让企业客户把礼品发放外包给平台：HR 只管发码，员工自己选尺码、填地址，平台负责履约。对 Cyndi 而言，这能显著减少短名单 Excel 往返，也能把一次性订单变成可复制的年度计划。

**功能清单**

1. 新表 `redemption_codes`（`code`、`portal_slug`、`points`、`status`、`opened_at`、`claimed_at`、`order_no`）与 `redemption_batches`（批次、生成数量、限额）。
2. 批量生成接口 `POST /api/v1/redemption_codes/create`，支持 `number_to_create`、`points`、单个码时附带 `email / first_name / last_name` 并自动邮件发送；设置日/年生成上限。
3. 状态查询接口 `GET /api/v1/redemption_codes/status`，状态取 `claimed / unclaimed / hidden`。
4. 兑换流程：兑换码作为 gate 登录凭证 → 员工在限定商品池内选品（可按尺码、颜色）→ 额度/积分校验 → 生成订单并回写 `order_no` → 扣减库存。
5. 与板块 F 的用户组预算联动，支持「每人 1 件」「每人 100 点」等规则。

**验收标准**：批量生成 100 个码并可对单个码查状态；员工兑换后自动生成订单、扣减对应 SKU 库存；码被使用后不可再次兑换。

**优先级 / 依赖**：P1；依赖 A、B、F 与 gate 会话体系。

### 5.6 板块 F｜企业客户身份与权限

**目标**：从「一个共享账号看全部商品」升级为「按客户、部门、用户组分配可见范围与预算」。

**为什么要做**：竞品提供 SSO、MFA、用户组与预算（Users, user groups & budgeting 分类）。Cyndi 当前的 gate 是单一共享访问方式，无法区分是谁下的单、谁用了多少预算，也无法对不同客户展示不同价格或商品池。

**功能清单**

1. 新表 `customer_accounts`、`customer_users`、`user_groups`、`budgets`、`approvals`；保留现有 gate 作为过渡入口。
2. 支持 SSO（客户身份提供商）与 MFA；会话 cookie 与商城、后台会话隔离，避免串号。
3. 权限规则：按用户组限定可见分类、商品价格可见性、单笔/累计额度、是否需要审批。
4. 审批流：超额或超范围订单进入待审批，审批通过后写入正式订单表。
5. 审计：登录、兑换、下单、审批均留操作人与时间。

**验收标准**：两个不同用户组登录后看到不同商品池；预算超限的订单被拦截并进入审批；共享账号关闭后不影响历史订单查询。

**优先级 / 依赖**：P1；依赖 gate 会话改造。

### 5.7 板块 G｜生态集成与批量导入

**目标**：把 Cyndi 的系统嵌入客户既有流程，降低下单门槛。

**为什么要做**：竞品 Integrations 分类中有 Workday（新员工/周年礼自动触发）、Salesforce App、Shopify 集成，并把 `csv` 列为正式订单来源。Cyndi 已有把客户 Excel 转为真实订单的内部脚本，把它产品化是成本最低的一步。

**功能清单**

1. CSV / Excel 批量下单：后台上传 → 表头映射 → 逐行校验（SKU、数量、MOQ、地址）→ 一次性返回全部错误行 → 确认后批量建单，来源记 `csv`。
2. 订单来源 `api` 与 `csv` 的对账视图（板块 D）。
3. 中期：Webhook，把订单状态、发货事件推送到客户系统。
4. 长期：HR 系统触发赠礼、CRM 商机礼品、电商平台店铺同步；按客户实际需求排期，不预先开发。

**验收标准**：一份含错误行的 Excel 上传后，错误行被完整列出且不产生任何订单；修正后可一键重试并全部成功。

**优先级 / 依赖**：P2；依赖 B、D。

### 5.8 板块 H｜对外文档中心

**目标**：把集成能力写成可交付、可销售、可自查的公开文档。

**为什么要做**：竞品这份页面本身就是销售资产——客户 IT 部门看完就能评估「能不能接」。Cyndi 目前没有对外文档，接口细节只存在于代码与内部记忆里，交付与排障成本高。

**功能清单**

1. 新建 `/docs` 或 `/integrations` 页面，栏目参考竞品知识库：Getting started、Authentication、SKU & inventory、Orders、Shipment status、Redemption codes、Errors & status codes、Changelog。
2. 每个端点给出：用途、请求示例、成功响应、失败响应、字段表、枚举值、限制说明、更新时间。
3. 提供沙盒申请与 API Key 获取流程说明（可暂由客户经理人工开通）。
4. 文档与接口版本绑定，接口变更同步更新 Changelog。

**验收标准**：文档示例可直接复制调用并返回一致结构；枚举值与后端常量一致；页面标注最后更新时间。

**优先级 / 依赖**：P2（可与 B 并行收尾）；依赖板块 B 定稿。

---

## 六、分阶段路线图

### 6.1 阶段划分

| 阶段 | 目标 | 包含板块 | 主要交付物 | 关键风险 |
|---|---|---|---|---|
| 阶段 0：数据地基 | 让每个商品与订单可被机器引用 | A、D 的字段部分 | SKU 编码规范、回填脚本、三张表新增字段、来源回填 | 历史订单与库存行迁移错误 |
| 阶段 1：对外可用 | 客户系统可查库存、下单、查追踪 | B、C、H | `/api/v1` 三个端点、API Key 管理、状态机、批次追踪、文档中心 | 绕过服务端定价、重复下单 |
| 阶段 2：规模化 | 从「人下单」转向「员工自助 + 客户系统集成」 | E、F、G、D 报表 | 兑换码 portal、用户组与预算、CSV 批量下单、对账报表 | 权限与 PII 合规 |

### 6.2 阶段 0 执行步骤

1. 输出《SKU 编码规范》一页纸，确定货号、色码、尺码三段规则与冲突处理。
2. 在 `inventory`、`order_items`、`orders` 增加 `sku_code`、`source`、`external_order_no` 字段；按项目约定同步 `CREATE TABLE` 与 `MIGRATIONS`。
3. 编写后端维护操作，一次性为缺 id 的颜色补 id 并回写 `colors_json`，再生成 SKU 编码；执行前导出备份。
4. 对 14 行期初库存、历史订单行做回填与一致性校验，输出「编码数 = 库存行数」的核对结果。
5. 后台库存页增加 SKU 编码列与搜索，验证颜色改名不引起编码变化。

### 6.3 阶段 1 执行步骤

1. 建 `api_keys` 与 `api_request_logs`，实现密钥创建、test / live 切换、吊销与轮换。
2. 实现 `/api/v1/skus/quantity`，只读、暴露可售量，明确「无行 = 按单生产」的返回语义。
3. 把现有下单逻辑抽成可被网页与 API 共用的服务端函数，实现 `/api/v1/orders/create` 与幂等约束。
4. 改造订单与 shipment 状态机，实现 `/api/v1/orders/status` 与批量查询。
5. 用 test key 跑端到端用例：查库存 → 建单 → 查状态 → 重复提交 → 吊销密钥，确认生产数据不被污染。
6. 发布文档中心，补齐错误码与枚举表。

### 6.4 阶段 2 执行步骤

1. 实现兑换码批量生成、状态查询与邮件发送，接入 gate 登录。
2. 实现用户组、预算与审批，替换共享账号访问方式。
3. 把内部 Excel 下单脚本产品化为后台 CSV 导入，错误行一次性反馈。
4. 上线按来源/日期的对账报表，与财务流程做一次完整勾稽演练。
5. 视客户需求评估 Webhook 与 HR / CRM / 电商集成，逐个立项。

---

## 七、不建议照搬的部分

| 竞品做法 | 不建议直接照搬的原因 | Cyndi 建议 |
|---|---|---|
| 拆成 `kwf.kotisdesign.com` 与 `stores.kotisdesign.com` 两个接口域 | 增加部署与证书复杂度，当前规模无收益 | 统一在 `www.cyndiglobal.com/api/v1/` 下 |
| 完整 JSON:API 规范 | 结构偏重，前端与后台需要额外适配 | 只借鉴 `data / relationships / included` 的分层思路，用普通 JSON 对象表达 |
| 积分（points）体系 | 若客户没有员工福利场景，积分会增加理解成本 | 先用「兑换码 + 件数/金额额度」，按需再引入积分 |
| 查询窗口硬限制 60 天 | 数据量小时会误伤月度/季度对账 | 设 90 天默认窗口 + 分页，超限提示缩小范围 |
| Live 密钥不可逆转 | 合理，但若没有轮换机制会影响故障处置 | 保留不可逆，同时提供 revoke / rotate 与审计日志 |
| 全部集成能力一次做齐 | Workday / Salesforce / Shopify 需要客户实际使用才值得投入 | 按客户需求单独立项，阶段 2 之后再看 |

---

## 八、风险与前置条件

1. **历史数据迁移**：SKU 编码与来源字段的回填涉及订单、库存、流水多张表，必须先备份、先在 test 环境演练、再执行；现有环境无 D1 直连 SQL 通道，所有变更需经后端维护接口完成。
2. **定价不能被绕过**：API 下单必须复用服务端定价与 MOQ 校验，客户端不得传单价、总价或创建时间；已有「改价自动重算 PENDING 订单」的逻辑要在 API 订单上同样生效。
3. **会话隔离**：API Key 与现有 showcase gate 会话、后台 `logistics_admin` 会话必须完全隔离；受保护响应统一 `Cache-Control: private, no-store`。
4. **前端发布纪律**：改前端须以远端仓库文件为基准做最小改动，本地工作区长期落后，禁止把本地旧文件整体推送；改 Functions 需完成语法检查、真实 import 校验、导入导出扫描与 mock D1 单测四步。
5. **PII 与合规**：兑换码邮件、收货地址、联系人属于个人信息，需最小化留存、限制后台可见范围，并在文档中说明用途。
6. **口径统一**：状态枚举、金额格式（千分位）、SKU 编码必须在网页、后台、接口、报表四处使用同一份常量，避免各写一套。

---

## 九、结论与下一步

竞品这页文档给出的最重要信号不是「他们有 8 个接口」，而是**他们把商品、订单、库存、履约、兑换、客户身份都做成了可被外部系统引用的数据对象**。Cyndi 已经把站内链路跑通，接下来要补的是「让这条链路对外开放」的八个板块。

建议本周先做三件不需要动线上页面的准备工作：

1. 定稿《SKU 编码规范》，明确 `货号-色码-尺码` 规则与历史数据回填方案；
2. 列出 `orders`、`inventory`、`shipments` 三张表需要新增的字段清单与迁移顺序；
3. 画出 `/api/v1` 三个最小端点的请求与响应样例，作为板块 B 的设计评审输入。

完成这三项后，即可进入阶段 1，把「客户系统能直接下单」变成可演示的能力。

---

## 附录 A｜Kotis 接口清单速查

| 方法 | 路径 | 用途 | 关键参数 |
|---|---|---|---|
| GET | `kwf.kotisdesign.com/api/v1/skus/quantity` | 查库存 | `Authorization` 头；返回 `sku`、`physical_quantity`、`available_quantity` |
| POST | `kwf.kotisdesign.com/api/v1/orders/create` | 建订单 | `email`、`shipping_method`、`order_id`、`skus`、`address` |
| GET | `kwf.kotisdesign.com/api/v1/orders/status` | 查订单与追踪 | `order_numbers`（逗号分隔） |
| GET | `kwf.kotisdesign.com/api/v1/orders/by_source` | 按来源查单 | `source`、`start_date`、`end_date`、可选 `portal_name` |
| POST | `stores.kotisdesign.com/api/v1/redemption_codes/create` | 生成兑换码 | `portal_slug`、`number_to_create`、可选 `points / email / first_name / last_name` |
| GET | `stores.kotisdesign.com/api/v1/redemption_codes/status` | 查兑换码状态 | `portal_slug`、`code` |

> 说明：页面中「Check order status」与「Get orders by source」「Create redemption codes」等章节均带有成功响应、失败响应与错误示例，本表仅作速查；完整字段以原文为准。

## 附录 B｜建议的状态与来源枚举（Cyndi 侧）

| 类别 | 建议取值 |
|---|---|
| 订单状态 | `pending`、`partially_shipped`、`shipped`、`delivered`、`cancelled` |
| 配送状态 | `unknown`、`pre_transit`、`in_transit`、`out_for_delivery`、`delivered`、`available_for_pickup`、`return_to_sender`、`failure`、`cancelled`、`error` |
| 订单来源 | `showcase_portal`、`admin_manual`、`api`、`csv`、`inventory_management` |
| 兑换码状态 | `unclaimed`、`claimed`、`hidden` |
| 密钥模式 | `test`、`live` |

## 附录 C｜资料来源与口径说明

1. 竞品信息来源于公开页面 `https://kotisdesign.forumbee.com/t/83hvjff/api-documentation` 及其分类页 `https://kotisdesign.forumbee.com/category/integrations`，抓取日期为 2026 年 11 月 13 日；页面标注更新时间为 2026 年 9 月 12 日。
2. 竞品接口域名、字段名、枚举值与错误示例均按原文转录；未对未公开的接口行为作推测。
3. Cyndi 现状依据项目内部系统记录（已上线页面、D1 表结构、订单与库存规则），不含尚未实施的规划内容。
4. 本文中的板块优先级、阶段划分与验收标准为规划建议，需结合实际客户合同与开发资源评审后确认。
