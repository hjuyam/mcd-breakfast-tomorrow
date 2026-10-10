<div align="center">

# 麦麦早餐管家 · mcd-breakfast-tomorrow

**明早那张 ¥9.9 的券，今晚就替你锁定了。**

你只需要回一个「好」。

![Type](https://img.shields.io/badge/Type-MCP%20Skill-DA291C?style=for-the-badge)
![Powered by mcd-mcp](https://img.shields.io/badge/Powered%20by-mcd--mcp-FFC72C?style=for-the-badge&labelColor=27251F)
![License](https://img.shields.io/badge/License-MIT-27251F?style=for-the-badge)

</div>

---

🔗 **在线演示（不用安装、不用 Token，点开就能看）** → https://hjuyam.github.io/mcd-breakfast-tomorrow/

---

## 这是什么

一个运行在支持 MCP 的 AI 客户端里的 Skill。它调用**麦当劳中国官方 MCP 服务**，替你把「明天早上的早餐」在**前一晚**就安排好：定好吃什么、算好用哪张券、建好预约单，然后把支付链接给你。

早上你到店，拿了就走。

> 本项目为麦当劳程序员节创意开发大赛参赛作品，由参赛者独立开发，**非麦当劳官方产品**。

---

## 为什么需要它

麦当劳的早餐券**按天发放，且只在当天早餐时段内有效**（通常 05:00–10:29）。

也就是说：你手上可能已经有明天早上那张 ¥9.9 的券，但它到点就作废；而早上你既没有时间研究该用哪张券，也来不及挑吃什么。

**真正的问题不是"早上吃什么"，而是这件事根本就不该在早上处理。**

---

## 它会做什么

| 能力 | 说明 |
|---|---|
| **学你的习惯** | 从你自己的历史订单里归纳：常去哪家店、几点吃、偏爱哪类主餐、固定搭配是什么、有没有常驻特制（例如"少冰""不要生菜"） |
| **算你的券** | 找出目标日有效的那张券，下单前复核它在当前门店、当前订单是否真的可用 |
| **查次日菜单** | 直接查询"明天早餐"的品类，而不是当前时段的菜单 |
| **出方案** | 给出 1–3 个候选，而不是把一整张菜单丢给你 |
| **建预约单** | 返回订单号与支付链接，**支付由你本人完成** |

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

### 3. 安装 Skill

把 `skills/mcd-breakfast-tomorrow/` 整个目录放进你客户端的 skills 目录（例如 WorkBuddy 的 `.workbuddy/skills/`）。

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
    今晚要锁定这一单吗？回「好」我就下单

你：好

它：订单已创建 ✅ 请在 15 分钟内完成支付
    🔗 https://...
    ⏰ 支付截止 14:52

（次日早晨）
它：取餐码 36031，到店出示即可
```

---

## 它是怎么知道该给你推什么的

**门店、口味、搭配、特制，全部由系统从使用者自己的历史订单中归纳——Skill 不预置任何个人偏好，也不区分用户。** 换成另一个人用，学出来的就是另一个人的习惯。

| 会归纳什么 | 从哪来 |
|---|---|
| 常用门店、出没时段 | 历史订单的门店与下单时间 |
| 主餐偏好 | 高频出现的主餐类别 |
| 固定搭配 | 反复出现的子项（通常是饮品或小食） |
| 常驻特制 | 逐单读取历史订单里的定制信息后归纳 |

---

## 支持与限制

- **支付必须由你本人完成。** 麦当劳官方不提供免密代付，本 Skill 只创建订单并返回支付链接。
- 餐品信息、价格及供应状态以**麦当劳官方渠道的实时结果**为准。
- 需要自备麦当劳 MCP Token；无 Token 时无法使用（可先看[在线演示](https://hjuyam.github.io/mcd-breakfast-tomorrow/)）。
- 当前默认**到店自取**，取餐时间需落在门店的早餐时段内。

---

## 仓库结构

```
mcd-breakfast-tomorrow/
├── README.md                       # 本文件
├── CONTEST_DECLARATION.md          # 参赛声明
├── MCP_INTEGRATION.md              # 实际调用的 MCP 工具、调用流程与参数说明
├── mcp-config.example.json         # 脱敏配置示例
├── workbuddy.md                    # WorkBuddy 开发对话上下文
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

- 本项目为麦当劳程序员节创意开发大赛参赛作品，由参赛者独立开发，**非麦当劳官方产品**。
- 本项目不构成对麦当劳及其关联方商标的任何授权。
- 仓库内不包含任何真实 Token、密钥或账号凭证；文中演示数据均为虚构。

---

## ⭐ 如果它让你的早上省了心

点个 **Star** 吧。这是麦当劳程序员节创意开发大赛的参赛作品，你的每一颗 Star 都算数。
