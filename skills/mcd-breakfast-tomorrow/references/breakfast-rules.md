# 早餐规则参考

本文件汇总本项目依赖的三类规则：**时段**、**券**、**特制**。所有结论均来自实测。

---

## 一、时段规则

| 项 | 规则 |
|---|---|
| 早餐供应时段 | 以门店返回的 `reservationTimeOptions` 为准。实测样本：`05:14–10:15` |
| 预约时段可选范围 | `query-nearby-stores` 的 `reservationTimeOptions` 会列出**次日**各时段；`today: true` 标记当天 |
| 跨时段查询 | `query-meals` 带 `reservationDate` 可拿到**次日早餐**品类——这是本项目成立的前提 |
| 目标取餐时间 | 必须落在该店早餐时段内，否则下单会失败 |
| 时间基准 | 一律以 `now-time-info` 的服务端时间为准 |

**实测对照（同一门店、同一时刻 14:08）**

| 调用 | 返回菜单 |
|---|---|
| 带 `reservationDate=2026-10-11 07:30` | 早餐（如麦满分系列、豆浆、薯饼） |
| 不带 | 午餐下午茶（巨无霸 / 随心配 1+1 / 500 大卡套餐 …） |

---

## 二、券规则

### 2.1 早餐券的特征

- **按天发放**：券绑定到具体某一天。
- **只在当天早餐时段有效**：实测样本的有效期为 `05:00–10:29`。
- **过期即作废**，不补发。

### 2.2 领券 → 筛券的顺序（不可颠倒）

券**不会自己出现在卡包里**。正确顺序：

```
available-coupons    （看当前有什么可领）
   → auto-bind-coupons   （一键领取全部可领券）
      → query-my-coupons    （取券包，按目标日筛出有效期的券）
         → query-store-coupons（当前可用，取 couponId / couponCode）
```

若跳过前两步直接查卡包，**卡包可能为空**，整套省钱逻辑落空。

### 2.3 四个接口的分工

| 接口 | 回答什么问题 | 关键字段 |
|---|---|---|
| `available-coupons` | 还能领什么 | 券名、状态、图片 |
| `auto-bind-coupons` | —（执行领券） | 无入参，一键全领 |
| `query-my-coupons` | 我持有什么、有效期到何时 | 券名、用券价格、有效期、标签 |
| `query-store-coupons` | 当前门店真正能用哪张 | **`couponId`**、**`couponCode`**、`tradeDateTime`、**`products[].productCode`** |

**要给订单挂券，必须拿到 `couponId` / `couponCode` —— 只有 `query-store-coupons` 提供。**

`query-my-coupons` **不承诺**券可用于当前订单。

### 2.4 ⚠️ 券的时刻可用性（关键实测）

`query-store-coupons` **按"当前时刻"判定券是否可用**，`reservationDate` 不会让它提前返回"未来才生效"的券。

**实测对照（同一账户、同一时刻 11:27，目标日 = 次日 07:30）**

| 接口 | 返回 |
|---|---|
| `query-my-coupons` | 12 张，含「次日的早餐券，有效期 次日 05:00–10:29」 |
| `query-store-coupons(reservationDate=次日 07:30)` | **1 张**（当日有效的那张），**未返回次日早餐券** |

**推论**：券只有在进入其有效期后，才能被 `query-store-coupons` 命中、才会给出 `couponId`/`couponCode`。

**因此建单要分流：**

| 情形 | 处理 |
|---|---|
| 券在目标日**已可用** | 直接按预约建单 |
| 券**尚未生效** | **不要提前建单**；落成方案，到**目标日早餐时段**再建单 |

> 提前建单会挂不上券、或按原价结算。

### 2.5 与门店/取餐方式的耦合

- 券常带"到店专用 / 外送专用"标签，须与取餐方式匹配。
- 券的有效期可能精确到"某天的某个时段"，跨天或跨时段都会失效。
- `query-store-coupons` 的 `products[]` 给出券的**适用商品编码** —— **券可以直接作为选品入口**。

---

## 三、特制规则

### 3.1 数据在哪

| 场景 | 位置 | 形态 |
|---|---|---|
| 历史订单（列表） | `order-list` | **无特制字段** |
| 历史订单（详情） | `query-order` → `comboItemList[].itemName` | **自然语言文本**，如 `吉士(不要)` |
| 可下单特制项 | `query-meal-detail` → `modification` / `rounds[].choices[].modification` | 结构化：`code` + `selectedKey` / `unselectedKey` |

**要从"历史习惯"走到"下单参数"，中间必须有一层文本→结构的映射。**

### 3.2 传参规则

- 特制项**成组回传**：该组内所有项都传；
- 选中的用 `selectedKey`，未选中的用 `unselectedKey`；
- 组内 `minValues=1 && maxValues=1` 且仅一项时**必选**，不可不选。

### 3.3 路径差异

| 场景 | 特制参数位置 |
|---|---|
| 单品 | `items[].modification` |
| **套餐** | `items[].roundList[].comboItemList[].modification` |

**两条路径不一样，套餐不能沿用单品的写法。**

### 3.4 常见特制类型

| 类型 | 例子 | 映射思路 |
|---|---|---|
| 酱 | 各主餐自带的酱类特制项 | 定位后取其 `unselectedKey` = 不要 |
| 芝士 | 吉士 | 定位吉士项，`unselectedKey` = 不要 |
| 冰量 | 标准 / 去冰 / 多冰 / 少冰 | 单选组，取其 `selectedKey` |
| 附加 | 奶油球 / 白砂糖 | 多选组，按需传入 |

### 3.5 容错

- 若某主餐已下架（`60001`），其特制映射随之失效 → 走同类替代或剔除候选；
- **宁可少带特制，也不可带错**：映射不确定时，不带特制建单，并明确提示用户自行确认。

---

## 四、其他价格规则

### 不要用实付金额推断价格偏好

- `order-list` 的 `realTotalAmount` 受券影响极大 —— 用券核销的订单可能为 **¥0**。
- **不能用它推断用户的价格偏好**，否则会把"用券的人"误判成"只买最便宜的人"。
- 价格判断一律用商品 `originalPrice` 或 `calculate-price` 的结果。

### 随单购早餐卡

- 商品 `discountType = "随单购早餐卡优惠"` 时，**必须同时选购早餐卡**才享该优惠价。
- 不满足条件时按 `originalPrice` 结算。
- **展示价格前必须判断**，不要让用户看到用不上的低价。

### 金额单位

- `calculate-price` 返回**分**，展示时除以 100。

### 取餐方式

- `calculate-price` 返回 `takeWayList[]`，实测样本：`eat-in` 堂食 / `take-in-store` 外带 / `take-dt-quick` 得来速。
- `create-order` 时 `orderType=1` 必须回传 `takeWayCode`。
