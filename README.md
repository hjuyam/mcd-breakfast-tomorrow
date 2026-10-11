<div align="center">

# 麦麦早餐管家 · mcd-breakfast-tomorrow

**明早那份早餐，今晚就替你安排好了。**

你只需要回一个「好」。

![Type](https://img.shields.io/badge/Type-MCP%20Skill-DA291C?style=for-the-badge)
![Powered by mcd-mcp](https://img.shields.io/badge/Powered%20by-mcd--mcp-FFC72C?style=for-the-badge&labelColor=27251F)
![License](https://img.shields.io/badge/License-MIT-27251F?style=for-the-badge)

</div>

---

🔗 **在线演示（不用安装、不用 Token，点开就能看）** → https://hjuyam.github.io/mcd-breakfast-tomorrow/

---

## 这是什么

一个运行在支持 MCP 的 AI 客户端里的 Skill。它调用**麦当劳中国官方 MCP 服务**，替你把「明天早上的早餐」在**前一晚**准备好：定好吃什么、用上哪张券、算好价钱，需要时再替你建单。

早上你到店，拿了就走。

> 本项目是基于**麦当劳中国官方 MCP 服务**制作的开源项目，**非麦当劳官方产品**。

---

## 为什么需要它

麦当劳的早餐券**按天发放，且只在当天早餐时段内有效**（通常 05:00–10:29）。

也就是说：你手上可能已经有明天早上那张 ¥9.9 的券，但它到点就作废；而早上你既没有时间研究该用哪张券，也来不及挑吃什么。

**真正的问题不是"早上吃什么"，而是这件事根本就不该在早上处理。**

---

## 它会做什么

| 能力 | 说明 |
|---|---|
| **帮你把券领齐** | 先查当前可领的券并一键领取，再挑出明天真正能用的那张 |
| **学你的习惯** | 从你自己的历史订单里归纳：常去哪家店、几点吃、偏爱哪类主餐、固定搭配是什么、有没有常驻特制（例如"不要芝士""不要生菜"） |
| **出方案** | 给出 1–3 个候选，而不是把一整张菜单丢给你 |
| **算清账** | 用券试算，把原价、优惠、应付一并算给你看 |
| **替你建单** | 返回订单号与支付链接，**支付由你本人完成** |
| **带上你的特制** | 有常驻特制（不要芝士、不要生菜之类）时，自动写进订单，不用每次重新交代 |

### 那张表

它会把你接下来几天的早餐排成一张表——哪天用哪张券、吃什么、多少钱、什么状态。

![本周早餐安排表](assets/weekly-plan.svg)

---

## 快速开始

### 1. 申请麦当劳 MCP Token

前往 [open.mcd.cn/mcp](https://open.mcd.cn/mcp)，手机号登录 → 控制台 → 激活，复制 MCP Token。

### 2. 配置 MCP Server

把 [`mcp-config.example.json`](./mcp-config.example.json) 的内容加入你客户端的 MCP 配置，并把 Token 换成你自己的：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer ${MCD_MCP_TOKEN}"
      }
    }
  }
}
```

> 配置完成后，在客户端的连接器管理里点一次**信任/启用**。

### 3. 安装 Skill

把 `skills/mcd-breakfast-tomorrow/` 整个目录放进你客户端的 skills 目录（例如 WorkBuddy 的 `~/.workbuddy/skills/`）。

### 4. 建一次档，然后说一句话

第一次使用时它会问几个问题（门店、取餐时间），其余偏好它会自己从你的历史订单里学。之后你只需要说：

```text
你：安排明早的早餐
```

---

## 使用示例

> 以下为演示数据，用于说明交互形态。

```text
你：安排一下明早的早餐

它：明早 07:45 · 深圳·科技园店 · 到店自取
    🍔 猪柳蛋麦满分组合（不要番茄酱）
    ☕ 冰美式中杯（少冰）
    🎟️ 用【早餐两件套】券
    💰 ¥9.9（原价 ¥18.5）

    按这个准备吗？

你：好

它：明早那份已经安排好了。
    这张券 05:00 才生效，我到那时替你下单——你早上回一个「下」就行。

（次日早晨）
它：订单已创建 ✅ 请在 15 分钟内完成支付
    🔗 https://...
    ⏰ 支付截止 07:00

它：取餐码 36031，到店出示即可
```

> **关于那一步「次日早晨」**：券只有在进入自己的生效时段后，才能挂到订单上。为了确保你那张券**真的用得上**，如果它还没生效，Skill 会在生效后再替你下单——而不是提前建一张用不上券的单。

---

## 它是怎么知道该给你推什么的

**门店、口味、搭配、特制，全部由系统从使用者自己的历史订单中归纳——Skill 不预置任何个人偏好，也不区分用户。** 换成另一个人用，学出来的就是另一个人的习惯。

| 会归纳什么 | 从哪来 |
|---|---|
| 常用门店、出没时段 | 历史订单的门店与下单时间 |
| 主餐偏好 | 高频出现的主餐类别 |
| 固定搭配 | 反复出现的子项（通常是饮品或小食） |
| 常驻特制 | 逐单读取历史订单里的定制信息后归纳 |

价格判断则以商品本身的**原价与券后价**为准，不依赖你过去实际付了多少钱。

---

## 支持与限制

- **支付必须由你本人完成。** 麦当劳官方不提供免密代付，本 Skill 只创建订单并返回支付链接。
- **券要在生效后才能挂单**。因此建单可能在券生效时进行，而不是立刻（见上方示例）。
- **餐品信息以官方实时结果为准**。价格、供应状态、菜单调整都以麦当劳官方渠道为准；已下架的餐品会被自动替换或剔除。
- **需要自备麦当劳 MCP Token**，并在客户端配置 `mcd-mcp` 连接器（连接器按客户端分别生效）。无 Token 时可先看[在线演示](https://hjuyam.github.io/mcd-breakfast-tomorrow/)。
- 当前默认**到店自取**，取餐时间需落在门店的早餐时段内。

---

## 仓库结构

```
mcd-breakfast-tomorrow/
├── README.md                       # 本文件
├── CONTEST_DECLARATION.md          # 项目声明
├── MCP_INTEGRATION.md              # 实际调用的 MCP 工具、调用流程与参数说明
├── mcp-config.example.json         # 脱敏配置示例
├── workbuddy.md                    # WorkBuddy 开发过程上下文
├── docs/
│   └── index.html                  # 在线演示站
├── assets/
│   └── weekly-plan.svg             # 「本周早餐安排表」示意图
└── skills/mcd-breakfast-tomorrow/
    ├── SKILL.md                    # Skill 主体
    ├── references/
    │   ├── mcd-tools.md            # 工具调用顺序与参数要点
    │   └── breakfast-rules.md      # 早餐时段、券、特制规则
    └── assets/
        ├── profile.template.yaml   # 长期画像模板
        └── day-plan.template.yaml  # 次日计划模板
```

---

## 相关文档

- **想知道它调了哪些 MCP 工具、怎么调的** → [`MCP_INTEGRATION.md`](./MCP_INTEGRATION.md)
- **想直接用** → [`skills/mcd-breakfast-tomorrow/SKILL.md`](./skills/mcd-breakfast-tomorrow/SKILL.md)

---

## 免责声明

- 本项目基于**麦当劳中国官方 MCP 服务**制作，**非麦当劳官方产品**。
- 本项目不构成对麦当劳及其关联方商标的任何授权。
- 餐品信息、价格及供应状态以麦当劳官方渠道的实时结果为准；本项目输出仅供参考。
- 仓库内不包含任何真实 Token、密钥或账号凭证；文中演示数据均为虚构。

---

## ⭐ 如果它让你的早上省了心

点个 **Star** 吧——你的每一颗 Star 都算数。
