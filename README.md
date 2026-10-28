<div align="center">

# Swag 平台 · 系统仿制规格与交接文档

**Kotis 竞品拆解 · 能力基线 · 落地差距分析**

对标美国定制周边礼品（Swag）服务商 **Kotis Design**，为其客户交付的「仓储管理系统 + 下单系统」做完整拆解，并给出我们自建平台的实施路径。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Spec](https://img.shields.io/badge/spec-v1.0-green.svg)](#)
[![Docs](https://img.shields.io/badge/docs-3%20%E4%BB%B6-orange.svg)](#)

</div>

---

## 这个仓库解决什么问题

我们要自建一套**给客户提供定制礼品下单与仓储履约**的系统。Kotis 是这个赛道做得最成熟的样本（美国市场、G2「Promotional Products Management」连续 11 个季度第一，客户含 AWS、Kraft Heinz、MongoDB、DigitalOcean、Bethesda）。

本仓库把 Kotis 的产品能力**逆向拆解成可执行的规格**，并对照我们现有的 `cyndi-global-website` 仓库，输出**差距分析 + 分阶段路线**，让任何一位接手人可以在半小时内知道「我们在做什么、现在到哪了、下一步做什么」。

---

## 目录结构

```text
.
├── README.md                       ← 你在这里：交接主入口
├── LICENSE
├── .gitignore
└── docs/
    ├── Kotis系统拆解与仿制规格书.html   ← 能力基线（16 章 28 表，可打印/转 PDF）
    ├── API契约与数据模型速查.md          ← 6 个 API 端点 + 数据模型 DDL
    └── 落地差距分析.md                   ← 现状 vs 目标 + 路线图（★ 接手先看这份）
```

---

## 三份文档怎么用

| 文档 | 什么时候看 | 能回答的问题 |
|---|---|---|
| **📌 落地差距分析** | **接手第一天** | 我们现在有什么、还差什么、怎么排期、哪些问题要先问业务方 |
| **📘 系统拆解规格书** | 做功能设计时 | Kotis 的每个功能长什么样、参数是什么、边界条件有哪些 |
| **📗 API 契约与数据模型** | 写代码时 | 对外接口怎么定义、表怎么建、状态枚举有哪些 |

---

## 核心结论速览

### 一、Kotis 的产品本质是「一套库存 + 无限门户」

Portal 只决定五件事：**谁能看、看到什么、什么价、怎么付、要不要审批**。所有门户共享同一个 SKU 库存池。这是整个产品最值得抄的结构。

三端子系统：

| 子系统 | 承载 | 职责 |
|---|---|---|
| **A. 客户门户** Storefront | `stores.kotisdesign.com/<portal>` | 浏览、选规格、下单、兑换、查物流 |
| **B. 运营后台** MyKotis | `mykotis.kotisdesign.com` | 报表、订单查询、库存下单、门户配置、审批、预算 |
| **C. 仓储履约** WMS | 内部系统（移动端扫码） | 收货 QA、盘点、拣货打包、发运、轨迹回传 |

### 二、技术栈（实测，非推测）

| 证据 | 观测值 | 结论 |
|---|---|---|
| 响应头 `X-Runtime` | `0.009225`（秒级） | **Ruby on Rails**（Rails 默认中间件） |
| Cookie 名 | `_depot_session` / `_my_kotis_session` | **两个独立 Rails 应用** |
| 响应头 `X-Request-Id` | UUID 格式 | `ActionDispatch::RequestId` |
| API 响应格式 | `{data, included, jsonapi:{version:"1.0"}}` | 严格 **JSON:API 1.0** |
| 前端 | 合作方 LeapOut 公开案例 | 自研 headless，底层以 **Shopify** 为 commerce engine |

> **给我们的启示**：Kotis 的技术选型是 2003 年起家的 Rails + Shopify 存量包袱。
> 我们是 Cloudflare 全栈（Pages + Functions + D1 + R2），**不应为了模仿而换栈**——
> 真正要学的是它的**数据模型**（库存双量、订单/发货分离、SKU 粒度定义）和**管控语义**（审批、预算、门户配置）。

### 三、三个最容易做错的地方

1. **库存双量必须分离**
   Kotis 的 `physical_quantity`（货架实数）与 `available_quantity`（可下单数）是两个字段。
   若合并存储，多门户并发下单必然超卖。→ **我们的 `product_skus` 目前完全没有库存字段，这是 P0。**

2. **财务口径锁定在发货侧**
   Kotis 明确声明 Sales Report **不能**用于季度销售汇总，季度 Portal Sales 必须取 Shipment Report 的 `Shipment Subtotal`——因为用户可中途取消或改单。
   → **我们的 `orders` 表目前只有一个 `total REAL`，无法对账。**

3. **尺码独立成 SKU**
   Kotis 的 T 恤 S/M/L/XL 算 4 个 SKU（直接推高仓储费），预组套合并为 1 个。
   → **我们的 `products → product_colors → product_skus` 三层结构已天然对齐，这块不用改。**

### 四、我们与目标的能力差距

| 维度 | 现状 | 目标 | 缺口 |
|---|---|---|---|
| 商品模型 | ✅ SPU → 颜色 → SKU | 同 | 无 |
| 下单 / 购物车 / 账户 | ✅ 完整 | 同 | 无 |
| 物流查询 | ✅ 有特色（带仓库实拍图） | 同 | 无 |
| **库存管理** | ❌ 无库存字段 | 双量 + 成本 + 调整留痕 + 在途 + 盘点 | **大** |
| **订单管控** | ❌ 无 | 审批流 + 预算组 + 多门户 + 兑换码 | **大** |
| **订单数据结构** | ⚠️ `items_json` JSON 快照 | 拆出订单行 / 发货单 / 金额分项 | **中** |
| 报表 | ❌ 无 | 13+ 张管理报表 + PII 脱敏 | 中 |
| 对外 API | ❌ 无 | 6 个端点 | 中 |

**详细逐项对照见 [`docs/落地差距分析.md`](docs/落地差距分析.md)。**

---

## 实施路线（详见差距分析 §5）

以「跑通完整业务闭环」为里程碑，而非按功能清单逐个补齐：

| 阶段 | 目标 | 关键产出 |
|---|---|---|
| **A 地基打平** | 无新功能，纯改造 | `inventory` 表、`order_lines` 拆表、`orders` 补字段、`shipments` 表 |
| **B 仓储闭环** | 单门户跑通库存到发运 | 库存管理后台、自助下单、拣货打包工作台、发货单与物流打通 |
| **C 管控能力** | 多门户 + 管控 | `portals` 表、上架配置、审批流、预算组、三张核心报表 |
| **D 规模化** | 对外服务与差异化 | 兑换码积分、自助 CSV 批量、PII 脱敏、客户 REST API、门户主题化 |

---

## 差异化机会

Kotis 的第三方评测明确指出几个弱点，**建议实现时直接超越**：

| Kotis 的做法 | 我们的做法 |
|---|---|
| CSV 下单必须严格匹配模板，且**人工发邮件给客服代提交** | 自助上传 + 行级校验 + 精确到行号/字段的报错 + 预览确认 |
| SSO 启用即覆盖全平台，**不提供测试模式** | 提供沙箱 IdP / 测试租户 |
| 打样审批是服务式人工流程，无版本控制工具 | 版本化工作流 + 图片区域标注 + 多轮留痕 |
| 门户配置依赖交付团队协助 | 自助品牌主题 + 权限模拟器（预览某人能看什么） |

另有一条值得直接照抄：**报表 PII 自动脱敏**（Ship-To / Attn / Address1 / Address2 / Email 打码，保留 City / State / Zip / Country）。

---

## 情报来源

**一手（Kotis 官方）**
[平台总览](https://kotisdesign.com/platform/) · [定价页](https://kotisdesign.com/pricing/) · [Swag Stores](https://kotisdesign.com/swag-stores) · [API 文档](https://kotisdesign.forumbee.com/t/83hvjff/api-documentation) · [Portal Types 对比](https://kotisdesign.forumbee.com/t/m1hnk4z/portal-types-comparisons) · [Program Service Policy](https://kotisdesign.forumbee.com/t/35hja55/program-service-policy-warehousing-fulfillment-and-delivery-standards) · [报表清单](https://kotisdesign.forumbee.com/t/p8hp599/what-reports-can-i-pull) · [帮助中心](https://kotisdesign.forumbee.com/)

**实测指纹**
`curl -I https://stores.kotisdesign.com/` → nginx · `X-Runtime` · `_depot_session` · `X-Request-Id`
`curl -I https://mykotis.kotisdesign.com/` → `_my_kotis_session` · `X-Runtime: 0.003423`

**第三方**
[LeapOut Digital 案例](https://www.leapoutdigital.com/enterprise-headless-shopify-philippines/)（Kotis 的 Shopify → headless 演进） · [RFP.wiki 对比](https://www.rfp.wiki/marketing/promotional-product-management-software/kotis-design/printfection) · [iTechGuides 评测](https://www.itechguides.com/products/kotis-swag-platform) · [TechYorker](https://techyorker.com/products/kotis-swag-platform)

---

## 待确认的业务问题

接手前建议向业务方确认，**这些问题直接影响表设计，事后改动成本高**：

1. 库存归属——Cyndi 自己的成品仓，还是客户寄存仓？
2. 履约模式——继续「按订单生产」，还是引入「提前备货到仓」？
3. 多门户是刚需，还是只需要「按客户配品牌主题」？
4. 是否收仓储费？按 SKU 数 / 托盘 / 订单量？
5. 是否需要开客户 REST API？
6. 混合币种（USD/CNY）下汇率在哪个时点锁定？
7. 审批人是 Cyndi 内部员工，还是客户的 HR/行政对接人？

> 完整列表与影响分析见 [`docs/落地差距分析.md` §7](docs/落地差距分析.md)。

---

## 已知信息缺口

以下内容官方未公开，若要精确复刻财务侧需另行确认：

- Fulfillment Billing Report 的完整字段清单与季度账单行项目明细
- 具体商品单价与仓储履约费单价（需销售报价）
- Mockup / Showcase 的详细交互规格（仅有演示视频）
- Cycle Count 的 WMS 实际界面（仅描述逻辑）
- MyKotis 完整页面清单与权限矩阵细节

规格书中标注为「估算」或「推断」的内容已单独标明，请勿直接作为商业承诺依据。

---

## 相关仓库

- [`Georgiazhu/cyndi-global-website`](https://github.com/Georgiazhu/cyndi-global-website) — 官网 + Swag 商城 + 物流平台（本规格的实施对象）
- [`Georgiazhu/Product-Upload-Tool`](https://github.com/Georgiazhu/Product-Upload-Tool) — 商品上货工具

---

## 许可

本仓库文档采用 [MIT License](LICENSE)。

文档中引用的 Kotis Design 公开资料版权归 Kotis Design LLC 所有，此处仅作竞品研究引用。
