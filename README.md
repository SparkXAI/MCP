# SparkX MCP

把 SparkX AI 的广告能力接进你自己的 AI Agent（Claude、ChatGPT 等），数据从此融入你的工作流——大白话查数据、做分析、算真账，还能和你自己的成本、利润、目标放在一起算。不用登录平台、不用导表、不用在系统之间来回切换。

**当前版本 v1.0.0（数据查询，只读）**：什么都能查、什么都不改。「管」和「建」能力将在后续版本陆续开放。

---

## v1.0.0 能帮你做什么

- **用自然语言查你的数据**——直接问「上周各 campaign 按 ACOS 排个序」「这个产品线最近 8 周的 TACOS 趋势」，免登录、免导表。
- **结合你自己的数据算真账**——把你的成本 / 毛利 / 目标交给 AI，让它拉广告花费：「按真实毛利，哪些 campaign 在亏钱——砍还是加？」这种需要把广告数据和你自己的业务数据合起来算的问题，平台单独算不出来。
- **沉淀你自己的玩法**——把常用问法存成模板，甚至设成每周一自动跑的周报 routine。

### 你能查什么（三类数据）

| 类别 | 内容 |
|------|------|
| **报表 / 效果数据** | 曝光、点击、花费、销售额、ACOS、ROAS、CTR、CVR、CPC 等；AI 托管口径指标；总销售额、TACOS、会话次数、Buy Box 等业务指标 |
| **实体配置 / 元数据** | 广告活动、广告组、投放、推广商品、ASIN、托管组、产品线信息等 |
| **操作日志** | 人工与 AI 的操作记录，可按操作者、动作类型、实体、时间窗筛选 |

---

## 支持的 AI 助手

以下 AI 助手均已完成接入测试、验收通过：

| AI 助手 | 类型 | 推荐 |
|---------|------|------|
| **Claude Code** | 海外 | ⭐ 海外优先推荐 |
| **ChatGPT** | 海外 | ⭐ 海外优先推荐 |
| **WorkBuddy** | 国内 | ⭐ 国内优先推荐 |
| Hermes | 国内 | |
| Cherry Studio | 国内 | |
| 扣子 Coze | 国内 | |
| OpenClaw | 国内 | |

---

## 快速开始（4 步）

1. **获取 token**——登录 SparkX 后台 → 账号菜单 → **MCP & Skills** → 新建 Token（离开页面后无法再次查看，请立即保存）。
2. **配置 MCP**——在你的 AI 助手里，一句话配好（见下方）。
3. **安装 Skills**——本仓库 [`skills/`](skills/) 目录，强烈推荐安装，显著降低查询出错率。
4. **开始查询**——用自然语言提问。

### 一句话配置（推荐）

在支持 AI 自主配置的客户端（Claude、ChatGPT Codex、Cherry Studio 等）里直接发送：

```text
帮我配置 SparkX MCP
URL：https://mcp.sparkx.cn/mcp
Token：<你的 token>
配置完调 get_user_authorized_context 验证。
```

返回你授权的店铺列表即为成功。各客户端手动配置、验证与故障排查见 **[安装说明 →](docs/installation.md)**

---

## Skills

MCP 的 Tool 决定 AI"能拿到什么数据"，Skill 决定 AI"把数据用得好不好"——**没有 Skills，MCP 查询的出错率会明显上升**。当前包含三个官方 Skill（当前版本 **1.0.0**，与 MCP Server 版本对应）：

| Skill | 对应 MCP Tool | 用途 |
|-------|--------------|------|
| [query-ads-performance](skills/query-ads-performance/) | `get_ads_perf` | 查询广告效果指标：花费、ACOS、ROAS、趋势、排名、同环比 |
| [query-entity-metadata](skills/query-entity-metadata/) | `get_entity_metadata` | 查询实体配置：广告活动 / 广告组 / 投放 / ASIN / 托管组的名称、状态、设置 |
| [query-operation-log](skills/query-operation-log/) | `get_operation_log` | 查询操作日志：人工与 AI 的调价、调预算、启停记录 |

**安装方式**（任选其一）：

- **Claude Code / Codex / Cursor 等**：把 [Releases](../../releases) 里的 Skills 包发给 AI，说一句"帮我把这三个 Skill 装上"。
- **Claude 网页版 / 桌面 App 等界面类助手**：设置 → Skills → 上传，逐个添加。

版本历史见 [CHANGELOG.md](CHANGELOG.md)。

---

## 示例提问

- 「昨天我账户下所有店铺表现如何？找出最需要关注的 5 个问题。」
- 「上周花费最高的 10 个 campaign，带 ACOS。」
- 「最近 30 天对比这几个产品线的表现，从广告结构和定向类型角度给优化建议。」
- 「过去 7 天有哪些 AI 自动调价？分别是为什么？」
- 「谁在什么时候改了这个 campaign 的预算？」

**提示词技巧**：说清时间范围、维度、指标、排序、Top N；点名店铺；一次一个意图，复杂需求拆开问。

---

## 当前版本边界

- **只读**：这一版不改账户、不下操作（写能力在后续版本）。
- **范围**：你能查的店铺，与你的 SparkX 账号（主 / 子账号）权限一致。
- **历史回溯**：约可查最近 15 个月。
- **非秒级实时**：与 SparkX AI 平台数据更新节奏一致；效果数据最细到天。

---

## 安全提示

> Token 决定你能查询哪些店铺和哪些数据，请妥善保管、勿外传。建议在 MCP 客户端配置中使用环境变量存放 token，并保持 local scope（仅当前项目可见）。
