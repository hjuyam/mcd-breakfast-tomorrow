# 工具调用参考 · mcd-mcp

本文件记录本项目实际用到的 10 个工具、调用顺序与实测踩坑结论。

---

## 调用顺序总览

```
now-time-info
  └─ query-nearby-stores
       └─ order-list ──► query-order × N        （学习惯）
            └─ query-my-coupons / available-coupons  （取券）
                 └─ query-meals(reservationDate)     （拉次日菜单）
                      └─ query-meal-detail           （解析特制与可替换项）
                           └─ calculate-price        （试算 + 取 takeWayCode）
                                └─ create-order      （建单，返回支付链接）
                                     └─ query-order / cancel-order
```

---

## 逐个要点

### `now-time-info`
- 无入参。返回服务端 `datetime`、`date`、`dayOfWeek`、`timezone`。
- **每次执行都先调**，用它推算"明天"，不要用本地时间。

### `query-nearby-stores`
- `beType`：`1` 到店自取 / `5` 得来速；`searchType`：`1` 收藏门店 / `2` 按位置搜索（需 `city` + `keyword`）。
- 返回含 **`reservation`** 与 **`reservationTimeOptions`**（按日期列出可预约时段）。
- `beType=1` 时门店无 `beCode`；`beType=5` 时有，后续调用必须带上。

### `order-list`
- 无入参。返回近期到店/外送订单。
- 字段：`orderId` / `createTime` / `storeCode` / `storeName` / `orderStatus` / `orderProductList[].{productCode, productName, quantity, comboItemList}` / `realTotalAmount`。
- ⚠️ **不含特制/定制字段。**

### `query-order`
- 入参：`orderId`。
- ⚠️ **定制信息在这里**：`orderProductList[].comboItemList[].itemName`，格式为**自然语言文本**，如 `板烧鸡腿堡 生菜(不要)`、`麦辣鸡腿汉堡 番茄酱(不要)`。
- 需要把文本解析后，映射回 `query-meal-detail` 的 `code` + `key` 才能用于下单。
- 同一接口还返回 `couponList`、`realTotalAmount`、`pickupCode`、`expirePayTime`。

### `query-meals`
- `storeCode` / `orderType` / `beType` 必填；**`reservationDate` 预约场景必传**。
- 返回 `categories`（分类+餐品 code）、`meals`（详情映射：`name` / `image` / `currentPrice` / `originalPrice` / `discountType` / `withOrder`）。
- 餐品图片为真实 URL（`menu-img.mcd.cn`）。

### `query-meal-detail`
- `code` 必填；`reservationDate` 预约场景必传。
- 返回 `rounds[]`（套餐选配轮次、可替换项与 `diffPrice`）、`modification.items[].values[]`（特制项，含 `selectedKey` / `unselectedKey`）。
- 若 `supportModify=true`，名称后需标注【可特调】。

### `query-my-coupons` / `available-coupons`
- `query-my-coupons`：持有券，含有效期与标签，**不承诺可用于当前订单**。
- 下单前用 `query-store-coupons` 复核门店/订单可用性。

### `calculate-price`
- 必填 `storeCode` / `orderType` / `beType`；`items[]` 含 `productCode` / `quantity` / `couponId|couponCode` / `modification` / `roundList`。
- 返回 `productPrice` / `discount` / `price`（**单位：分**）与 **`takeWayList[].code`**。
- 实测：带 `reservationDate` 试算成功。

### `create-order`
- 与 `calculate-price` 参数基本一致，外加 **`takeWayCode`**（`orderType=1` 必传）。
- 返回订单号与**支付链接**；官方不提供免密代付。

### `cancel-order`
- 入参 `orderId` + `cancelReasonCode`（`1` 改主意了 / `2` 重复下单 / `3` 点错 / `4` 地址电话错 / `5` 送达时间错 / `-1` 其它）。

---

## 实测踩坑记录

| 现象 | 结论 |
|---|---|
| 同店同时刻，带与不带 `reservationDate` 返回**完全不同的菜单** | `reservationDate` 是查询维度，不只是"预约标记" |
| `order-list` 看不到定制，`query-order` 才看得到 | 学习惯必须**逐单**拉详情 |
| 定制是文本（`吉士(不要)`）而非结构化字段 | 需要一个"文本 → code/key"的映射层 |
| 部分商品 `discountType = 随单购早餐卡优惠` | 必须同时买卡，否则按原价，展示价会误导 |
| 某餐品返回 `60001 未匹配到商品` | SKU 会动态下架，映射与推荐都要有降级 |
| 门店返回 `reservation: true` 并列出次日早餐时段 | 跨时段预约是官方原生能力 |
