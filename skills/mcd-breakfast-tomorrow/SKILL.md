---
name: mcd-breakfast-tomorrow
description: 前一晚替用户锁定明早的麦当劳早餐。读取常点习惯与当日有效的早餐券，取交集生成方案，并预约建单返回支付链接。当用户提到"安排明早的早餐""明早吃什么""锁定明天的早餐券""帮我订明天的麦当劳早餐""早餐提前订"时使用本技能。
---

# 麦麦早餐管家 · mcd-breakfast-tomorrow

**核心价值：早上零动作。** 决策与下单都发生在用户睡前。

> 本 Skill 只负责编排业务逻辑，本身不含 MCP 连接。Token 由用户在客户端 connector 侧配置，**绝不写入本文件或任何交付物**。

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
| 3 | **券要按"目标日"筛** | 早餐券按天发放，且**只在当天 05:00–10:29 有效**。必须筛出"目标日在有效期内"的券，而不是"用户持有的所有券"。 |
| 4 | **习惯优先于推荐** | 先取交集，再考虑变化。不要推荐用户从不吃的主餐类别。 |
| 5 | **特制必须带上** | 用户有常驻特制（如去酱、去芝士）时，**必须写入下单参数**，否则做出来的是错的。 |
| 6 | **时间要对齐服务端** | 每次执行先调 `now-time-info`，用它判断"明天"是哪一天。 |
| 7 | **个人信息不外发** | 画像与计划属个人信息，只存本地，不写入仓库或任何对外交付物。 |

---

## 数据文件

### `profile.yaml`（长期画像，建档时生成）

见 `assets/profile.template.yaml`。

### `plans/YYYY-MM-DD.yaml`（次日计划，每晚一份）

见 `assets/day-plan.template.yaml`。`status` 字段是"执行到哪一步"的唯一事实来源：

`drafted` → `confirmed` → `ordered` → `paid` → `done`

---

## 工作流

### Step 0 · 建档（首次）

1. `now-time-info` 对齐服务端时间；
2. `query-nearby-stores`（`beType=1`）→ 取用户选择的 `storeCode`，记录 `reservation` 与 `reservationTimeOptions`；
   - 若 `reservation=false`，明确告知该门店不支持预约，询问是否换店；
3. 记录取餐方式（默认 `take-in-store` 外带）与目标取餐时间（默认 07:30，须落在该店早餐时段内）；
4. 生成 `profile.yaml`。

### Step 1 · 学习惯

1. `order-list` 取历史订单；
2. **逐单** `query-order` 读 `comboItemList[].itemName` —— 定制信息在这里，且是自然语言文本；
3. 归纳并写入画像：
   - **高频门店** → 与建档门店互相印证
   - **出没时段** → 决定默认取餐时间
   - **主餐偏好** → 从高频 `productName` 中提取类别（如"鸡扒类"）
   - **固定搭配** → 反复出现的子项（如"优品豆浆小杯"）
   - **常驻特制** → 反复出现的 `(不要)` 项（如 `去酱` / `去芝士`）
4. 把归纳结果**回读给用户确认**（一句话），不要默认正确。

### Step 2 · 取券

1. `query-my-coupons` 取券包，过滤出**目标日在有效期内**的券；
2. 需要时用 `available-coupons` 补齐"可领取"的券；
3. 下单前用 `query-store-coupons` 复核该券在当前门店/订单是否可用。

### Step 3 · 拉次日菜单

`query-meals`：

```
storeCode       = <建档门店>
orderType       = 1
beType          = 1
reservationDate = "YYYY-MM-DD HH:mm"   # 目标日 + 取餐时间
```

返回的即为**目标日早餐品类**。

### Step 4 · 取交集并试算

1. 候选池 = `主餐偏好 ∧ 券适用商品`；
2. 对候选用 `query-meal-detail` 解析：
   - 套餐的可替换项（`rounds[].choices[]`，含 `diffPrice`）
   - 主餐的特制项（`code` / `selectedKey` / `unselectedKey`）
3. 套用画像中的**常驻特制**（如去酱 → 该特制项用 `unselectedKey`）；
4. `calculate-price` 试算，拿到金额、优惠与 `takeWayList[].code`；
5. 生成方案：**1–3 个候选**，不是整张菜单。

### Step 5 · 确认与建单

1. 向用户展示方案（餐品 + 特制 + 用哪张券 + 券后价 + 取餐时间）；
2. 用户确认后 `create-order`：

```
storeCode       = <建档门店>
orderType       = 1
beType          = 1
reservationDate = "YYYY-MM-DD HH:mm"
takeWayCode     = <来自 calculate-price 的 takeWayList[].code>
items[]         = { productCode, quantity, couponId/couponCode, modification, roundList }
```

3. 返回**订单号 + 支付链接 + 支付截止时间**，并明确说明：**支付需本人完成**；
4. 更新 `plans/YYYY-MM-DD.yaml` 的 `status` 为 `ordered`。

### Step 6 · 次日晨

- 用户告知已支付 → `query-order` 复核状态 → 更新 `status = paid`；
- 输出取餐信息（`pickupCode` / `lockerCode`）。

---

## 关键参数规则

### `reservationDate`

- 格式：`yyyy-MM-dd HH:mm`
- `query-meals` / `query-meal-detail` / `calculate-price` / `create-order` **四者都要带**，否则会被当作"当前时段下单"。

### `modification`

- 特制项必须**成组回传**：该组内所有项都传，选中的用 `selectedKey`，未选中的用 `unselectedKey`；
- 组内 `minValues=1 && maxValues=1` 且仅一项时为必选，不可不选；
- **套餐的特制在 `roundList[].comboItemList[].modification`，与单品路径不同。**

### `withOrder`（随单购早餐卡）

商品若标 `discountType = "随单购早餐卡优惠"`，**必须同时选购早餐卡**才享优惠价，否则按原价结算。
→ 展示价格前必须判断：不满足条件时，**按原价展示**，不要让用户看到用不上的低价。

### `takeWayCode`

`orderType=1` 必传，取值来自 `calculate-price` 返回的 `takeWayList[].code`。

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
- 定时唤起**默认关闭**。需要时由用户显式开启，并带"明天不用"出口。
