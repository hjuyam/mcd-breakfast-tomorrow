---
name: mcd-breakfast-tomorrow
description: 前一晚替用户把明早的麦当劳早餐准备好。读取常点习惯与早餐券，取交集生成方案，并在券可用时预约建单、返回支付链接。当用户提到"安排明早的早餐""明早吃什么""锁定明天的早餐券""帮我订明天的麦当劳早餐""早餐提前订"时使用本技能。
---

# 麦麦早餐管家 · mcd-breakfast-tomorrow

**核心价值：不用想、不用选——明早那一单，前一晚就替你准备好了。**

> 本 Skill 只负责编排业务逻辑，本身不含 MCP 连接。Token 由用户在客户端 connector 侧配置，**绝不写入本文件或任何交付物**。

---

## 通用性声明

本 Skill 面向**任意用户**，不预置任何个人偏好：

- 门店、出没时段、主餐偏好、固定搭配、常驻特制，**全部在运行时从使用者自己的历史订单中归纳**；
- 需要人工给出的信息（门店、取餐时间）一律在建档时询问，**不写死**；
- `assets/*.template.yaml` 中的示例值均为占位，**不含任何真实用户数据**。

> 换言之：换一个用户使用，学出来的就是另一个人的习惯。本 Skill 的通用性不依赖任何特定账号。

---

## 何时使用

用户表达"把明早的早餐提前办好"这类意图时：

- "安排明早的早餐" / "明早吃什么"
- "锁定明天那张早餐券"
- "帮我订明天的麦当劳早餐"

**不适用**：用户只是想看看菜单、想当场点餐、想问营养信息 → 直接调用对应工具回答即可，不走本流程。

---

## 前置条件

1. 客户端已配置 `mcd-mcp` connector 且 Token 有效；
2. 用户已完成一次建档（或本次对话中完成）。

---

## 核心原则（必读，违反即出错）

| # | 原则 | 说明 |
|---|---|---|
| 1 | **支付必须人工** | `create-order` 只返回订单号与支付链接。**永远不要说"已帮你订好早餐"**，只能说「订单已创建，请在 X 分钟内完成支付」。 |
| 2 | **锁定门店** | 不使用"附近的店"动态选择。门店的菜单、价格、早餐起供时间都不同。`storeCode` 在建档时写死。 |
| 3 | **券要先领后查** | 券不会自己出现在卡包里。**必须先 `available-coupons` → `auto-bind-coupons`，再 `query-my-coupons`**。顺序颠倒会导致卡包为空、整套省钱逻辑落空。 |
| 4 | **券按"目标日"筛，按"当前时刻"判可用** | 早餐券通常限定「某天 05:00–10:29」。**筛选**按目标日；**能否挂到订单上**取决于券是否已进入可用期——因此可能需要**延迟建单**（见 Step 5）。 |
| 5 | **习惯优先于推荐** | 先取交集，再考虑变化。不要推荐用户从不吃的主餐类别。 |
| 6 | **特制必须带上** | 用户有常驻特制（例如"不要芝士""不要生菜"）时，**必须写入下单参数**，否则做出来的是错的。 |
| 7 | **时间要对齐服务端** | 每次执行先调 `now-time-info`，用它判断"明天"是哪一天。 |
| 8 | **不要用实付金额推断价格偏好** | `realTotalAmount` 受券影响极大（用券核销的订单可能为 ¥0）。价格判断一律用商品 `originalPrice` 或 `calculate-price` 结果。 |
| 9 | **个人信息不外发** | 画像与计划属个人信息，只存本地，不写入仓库或任何对外交付物。 |
| 10 | **可被定时任务调用** | 本 Skill 可能由定时任务在无人值守时触发。此时输出必须**自成一体**（一条可直接推送的短消息），不要依赖后续追问。 |

---

## 数据文件

### `profile.yaml`（长期画像，建档时生成）

见 `assets/profile.template.yaml`。**属可选缓存**：若不存在，Step 1 会重新推导，不影响运行。

### `plans/YYYY-MM-DD.yaml`（次日计划）

见 `assets/day-plan.template.yaml`。`status` 字段是"执行到哪一步"的唯一事实来源：

```
planned → confirmed → ordered → paid → done
```

> `planned` = 方案已就绪、但**尚未建单**（券还没到可用期）。见 Step 5 情形 B。

---

## 工作流

### Step 0 · 建档（首次）

1. `now-time-info` 对齐服务端时间；
2. `query-nearby-stores`（`beType=1`）→ 取用户选择的 `storeCode`，记录 `reservation` 与 `reservationTimeOptions`；
   - 若 `reservation=false`，明确告知该门店不支持预约，询问是否换店；
3. 记录取餐方式（默认 `take-in-store` 外带）与目标取餐时间（须落在该店早餐时段内）；
4. **门店与取餐时间可先从历史订单推断，再请用户确认**，以减少提问；
5. 生成 `profile.yaml`。

### Step 1 · 学习惯

1. `order-list` 取历史订单；
2. **逐单** `query-order` 读 `comboItemList[].itemName` —— 定制信息在这里，且是自然语言文本；
3. 归纳并写入画像（下列示例仅说明归类方式，非任何真实用户数据）：
   - **高频门店** → 与建档门店互相印证
   - **出没时段** → 决定默认取餐时间
   - **主餐偏好** → 从高频 `productName` 中提取类别（例如"牛肉堡类"）
   - **固定搭配** → 反复出现的子项（例如"中杯可乐"）
   - **常驻特制** → 反复出现的定制项（例如"不要芝士""不要生菜"）
4. ⚠️ **不要用 `realTotalAmount` 推断价格偏好**（见核心原则 8）；
5. 把归纳结果**回读给用户确认**（一句话），不要默认正确。

### Step 2 · 领券 → 筛券（顺序不可颠倒）

1. `available-coupons` —— 看当前有什么可领的券；
2. `auto-bind-coupons` —— **一键领取全部可领券**（默认执行；结束后用一句话告知领到了什么）；
3. `query-my-coupons` —— 取券包，筛出**目标日在有效期内**的券；
   - 筛选维度是**目标日**，不是"当前时刻"（当前时刻用于 Step 4 判可用性）；
4. 记录每张候选券的名称、用券价格与有效期。

### Step 3 · 拉次日菜单

`query-meals`：

```
storeCode       = <建档门店>
orderType       = 1
beType          = 1
reservationDate = "YYYY-MM-DD HH:mm"   # 目标日 + 取餐时间
```

**用途**：确认券对应商品**当前仍在售**（SKU 会动态下架）。

### Step 4 · 取交集并试算

1. 候选池 = `主餐偏好 ∧ 券适用商品`；
2. 对候选用 `query-meal-detail`（带 `reservationDate`）解析：
   - 套餐的可替换项（`rounds[].choices[]`，含 `diffPrice`）
   - 主餐的特制项（`code` / `selectedKey` / `unselectedKey`）
3. 套用画像中的**常驻特制**；
4. `query-store-coupons`（带 `reservationDate`）→ 取券的 **`couponId` / `couponCode`**，以及 `products[]`（券适用商品）；
   - ⚠️ 若**取不到目标日的早餐券**，说明该券在其有效期内尚未生效 → 走 Step 5 情形 B；
5. `calculate-price`（带 `reservationDate` + `couponId`/`couponCode`）试算，取回金额与 `takeWayList[].code`；
6. 生成方案：**1–3 个候选**，不是整张菜单。

### Step 5 · 建单（按券的可用性分流）

**情形 A —— 券在目标日已可用**（Step 4.4 已拿到 `couponId`/`couponCode`）：

用户确认后 `create-order`：

```
storeCode       = <建档门店>
orderType       = 1
beType          = 1
reservationDate = "YYYY-MM-dd HH:mm"
takeWayCode     = <来自 calculate-price 的 takeWayList[].code>
items[]         = { productCode, quantity, couponId, couponCode, modification, roundList }
```

→ 返回订单号 + 支付链接 + 支付截止时间，明确说明**支付需本人完成**。`status = ordered`。

**情形 B —— 券尚未进入可用期**（Step 4.4 取不到券的 id/code）：

**不要提前建单。** 把方案落成 `status = planned`，并告知用户：

> 明早那份已经安排好了：<餐品>，用【<券名>】券 ¥X.XX。
> 这张券要到明早 <时段> 才生效，所以我到那时再帮你下单——你早上回一个「下」就行。

到**目标日早餐时段**，由定时任务或用户一句话触发，重新执行 Step 4.4 → 4.5 → 情形 A。

> 这一分流是必要的：**券必须在其有效期内才可挂到订单上**。提前建单会挂不上券、或按原价结算。

### Step 6 · 次日晨

- 建单后推送**订单号 + 支付链接 + 支付截止时间**；
- 用户告知已支付 → `query-order` 复核状态 → `status = paid`；
- 输出取餐信息（`pickupCode` / `lockerCode`）。

---

## 关键参数规则

### `reservationDate`

- 格式：`yyyy-MM-dd HH:mm`
- `query-meals` / `query-meal-detail` / `query-store-coupons` / `calculate-price` / `create-order` **都要带**，否则会被当作"当前时段下单"。
- ⚠️ 实测：`query-store-coupons` **按"当前时刻"判定券是否可用**，`reservationDate` 不会让它提前返回"未来才生效"的券。

### `modification`

- 特制项必须**成组回传**：该组内所有项都传，选中的用 `selectedKey`，未选中的用 `unselectedKey`；
- 组内 `minValues=1 && maxValues=1` 且仅一项时为必选，不可不选；
- **套餐的特制在 `roundList[].comboItemList[].modification`，与单品路径不同。**

### 券

```
available-coupons（可领）
  → auto-bind-coupons（领）
    → query-my-coupons（持有，按目标日筛）
      → query-store-coupons（当前可用，取 couponId / couponCode + products[]）
```

- `query-store-coupons` 返回 `products[].productCode` —— **券自带适用商品，可作为选品入口**；
- 券若带"到店专用 / 外送专用"标签，须与取餐方式匹配。

### `withOrder`（随单购早餐卡）

商品若标 `discountType = "随单购早餐卡优惠"`，**必须同时选购早餐卡**才享优惠价，否则按原价结算。
→ 展示价格前必须判断：不满足条件时，**按原价展示**，不要让用户看到用不上的低价。

### 金额单位

`calculate-price` 返回**分**，展示时除以 100。

---

## 异常处理

| 情况 | 处理 |
|---|---|
| 门店 `reservation=false` | 告知不支持预约，询问是否换店；不静默降级 |
| 目标日菜单查不到 | 降级为"当天早晨执行"方案，并说明原因 |
| 餐品下架（`60001 未匹配到商品`） | 用同类替代；无替代则从候选中剔除并说明 |
| 特制映射缺失 | 不带特制建单，并**明确提示用户自行确认**（宁可少带，不可带错） |
| 券在目标日尚未生效 | 走 Step 5 情形 B，**不提前建单** |
| 试算超预算 | 不下单，给出"换组合"选项 |
| 建单成功但未支付 | 提醒一次；仍不支付则 `cancel-order` 释放 |
| 用户不回应 | 按 `profile.yaml` 的 `no_reply` 策略处理，**默认 `remind_only`（什么都不做）** |

---

## 输出规范（措辞红线）

**禁止**：

- ❌「已帮你订好早餐」
- ❌「明天早上的早餐搞定了」

**必须**：

- ✅「订单已创建，请在 X 分钟内完成支付」
- ✅「支付需你本人完成，我这边只能把订单准备好」

这个措辞差异直接影响用户对订单状态的判断——**未支付的订单不等于早餐保住了**。

---

## 表达与交互

- 方案要**短**：1–3 个候选，每个一行（餐品 · 特制 · 券 · 价格）。
- 用户的动作应是**选择题**（确认 / 换一个 / 微调），不是填空题。
- 用户说"不用了"时，**零成本退出**：不追问、不统计、不补推。
- 无人值守触发时，输出必须**一条消息讲完**（方案 + 券 + 价格 + 需要用户做什么）。
