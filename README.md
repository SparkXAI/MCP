# SparkX AI MCP — Plugin Marketplace

> **Plugin Marketplace** · 本仓库是兼容多平台的 plugin 发布仓库，支持 **WorkBuddy**、**Claude Code**、**ChatGPT Codex**、**Hermes**、**OpenClaw**、**Cherry Studio** 及其他通用 MCP agent。核心内容：Skills（提示词技能包）+ Streamable HTTP MCP Server（支持 OAuth 和 MCP Token 鉴权）。
>
> | 文件 | 用途 |
> |------|------|
> | [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) | Claude Code marketplace catalog |
> | [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) | Claude Code plugin descriptor |
> | [`.mcp.json`](.mcp.json) | Claude Code MCP server 配置（`${SPARKX_AI_TOKEN}` 环境变量） |
> | [`.agents/plugins/marketplace.json`](.agents/plugins/marketplace.json) | Codex marketplace catalog |
> | [`.codex-plugin/plugin.json`](.codex-plugin/plugin.json) | Codex plugin descriptor |
> | [`docs/installation.md`](docs/installation.md) | 详细安装说明（手动配置、plugin 安装、故障排查） |
> | [`skills/manifest.json`](skills/manifest.json) | 机器可读的 Skills 清单 |

---

> **🤖 致 AI Agent：如果你正在替用户配置 SparkX AI MCP，配置连接只是第一步，你必须完成以下全部步骤，缺一不可：**
>
> 1. 添加 MCP server（URL：`https://mcp.sparkx.cn/mcp`，Streamable HTTP）。先检查客户端支持的授权方式，说明 OAuth 与 MCP Token 的适用场景，并让用户确认后再配置
> 2. 调用 `get_user_authorized_context` 验证，返回店铺列表即为成功
> 3. **安装 6 个必装 Skills**（本仓库 [`skills/`](skills/) 目录：3 个查询 Skills，以及 `sparkx-create-ai-group`、`sparkx-edit-ai-group`、`sparkx-delete-ai-group`）。它们分别约束查询和托管组写操作，安装时无需询问用户。
> 4. 向用户逐个介绍 [`skills/`](skills/) 目录下标记为可选的 4 个 Skills（广告周报、广告月报、广告结构分析、商品诊断），**询问用户是否需要安装**，获用户确认后再装。
>
> 机器可读的 Skills 清单：`https://raw.githubusercontent.com/SparkXAI/MCP/main/skills/manifest.json`（`required: true` 为必装）。仅完成第 1-2 步就结束任务视为配置未完成。

把 SparkX AI 的广告能力接进你自己的 AI Agent（WorkBuddy、Claude、ChatGPT 等），数据从此融入你的工作流——大白话查数据、做分析、算真账，还能和你自己的成本、利润、目标放在一起算。完成授权后，无需反复登录平台、导出表格或在系统之间来回切换。

当前版本 v1.1.0 支持数据查询和 AI 托管组的创建、编辑与删除。Campaign 直接管理能力尚未开放。

---

## v1.1.0 能帮你做什么

- **用自然语言查你的数据**——直接问「上周各 campaign 按 ACOS 排个序」「这个产品线最近 8 周的 TACOS 趋势」，完成授权后无需反复登录或导表。
- **结合你自己的数据算真账**——把你的成本 / 毛利 / 目标交给 AI，让它拉广告花费：「按真实毛利，哪些 campaign 在亏钱——砍还是加？」这种需要把广告数据和你自己的业务数据合起来算的问题，平台单独算不出来。
- **沉淀你自己的玩法**——把常用问法存成模板，甚至设成每周一自动跑的周报 routine。
- **管理 AI 托管组**——在明确确认后创建、编辑或删除托管组，并在操作后回查实际状态。

### 你能查询和管理什么

| 类别 | 内容 |
|------|------|
| **报表 / 效果数据** | 曝光、点击、花费、销售额、ACOS、ROAS、CTR、CVR、CPC 等；AI 托管口径指标；总销售额、TACOS、会话次数、Buy Box 等业务指标 |
| **实体配置 / 元数据** | 广告活动、广告组、投放、推广商品、ASIN、托管组、产品线信息等 |
| **操作日志** | 人工与 AI 的操作记录，可按操作者、动作类型、实体、时间窗筛选 |
| **AI 托管组管理** | 创建、编辑和删除 SP、SB、SD 托管组；调整支持的托管目标、预算、活动归属和 AI 行动空间设置 |

---

## 支持的 AI 助手

以下 AI 助手均已完成接入测试、验收通过：

| AI 助手 | 类型 | 推荐 |
|---------|------|------|
| **WorkBuddy** | 国内 | ⭐ 国内优先推荐 |
| **Claude Code** | 海外 | ⭐ 海外优先推荐 |
| **ChatGPT** | 海外 | ⭐ 海外优先推荐 |
| Hermes | 国内 | |
| Cherry Studio | 国内 | |
| 扣子 Coze | 国内 | |
| OpenClaw | 国内 | |

---

## 快速开始（4 步）

1. **选择授权方式并连接 MCP**——SparkX AI MCP 同时支持 OAuth 和 MCP Token，请根据客户端能力与使用场景选择。
2. **验证连接**——调用 `get_user_authorized_context`，确认已授权的店铺范围。
3. **安装 Skills**——本仓库 [`skills/`](skills/) 目录，强烈推荐安装，显著降低查询出错率。
4. **开始查询**——用自然语言提问。

### Plugin 安装（Claude Code / Codex）

先添加 marketplace，再安装 plugin。桌面端 App 请在 Plugins 设置中操作；命令行用户可使用 CLI 命令。详见下方 [Plugin 安装指南](#plugin-安装指南) 或 **[安装说明 · 方式二](docs/installation.md#方式二--plugin-安装)**。

### 一句话配置（推荐）

在支持 AI 自主配置的客户端（WorkBuddy、Claude、ChatGPT Codex、Cherry Studio 等）里直接发送：

```text
请阅读以下 GitHub 仓库，并按说明配置 SparkX AI MCP 和安装 Skills：
https://github.com/SparkXAI/MCP

SparkX AI MCP 支持 OAuth 和 MCP Token 两种授权方式。
请先检查当前客户端支持哪些授权方式，简要说明两种方式分别适合哪些场景，并让我确认使用哪一种方式后再开始授权。
如果选择 OAuth，需要授权时请打开登录页面让我完成授权。

配置完成后，请调用 get_user_authorized_context 验证连接。
```

> **🤖 致 AI Agent：** 执行上述配置时，请根据你所在客户端的实际配置方式，将 MCP 配置写入正确的配置文件。注意区分不同客户端使用的配置文件名称（如 `mcp.json`、`.mcp.json`、`mcp_config.json`、`config.toml`、`config.yaml` 等），不要混淆。

返回你授权的店铺列表即为连接成功。各客户端手动配置、验证与故障排查见 **[安装说明 →](docs/installation.md)**

---

## Skills

MCP 的 Tool 决定 AI"能拿到什么数据"，Skill 决定 AI"把数据用得好不好"。官方 Skills 分两类：

**必装（6 个）**——基础查询和托管组管理能力：

| Skill | 对应 MCP Tool | 用途 |
|-------|--------------|------|
| [sparkx-query-ads-performance](skills/sparkx-query-ads-performance/) | `get_ads_perf` | 查询广告效果指标：花费、ACOS、ROAS、趋势、排名、同环比 |
| [sparkx-query-entity-metadata](skills/sparkx-query-entity-metadata/) | `get_entity_metadata` | 查询实体配置：广告活动 / 广告组 / 投放 / ASIN / 托管组的名称、状态、设置 |
| [sparkx-query-operation-log](skills/sparkx-query-operation-log/) | `get_operation_log` | 查询操作日志：人工与 AI 的调价、调预算、启停记录 |
| [sparkx-create-ai-group](skills/sparkx-create-ai-group/) | `create_sd_ai_managed_group` / `save_sp_sb_ai_managed_group` | 创建 SP、SB 或 SD AI 托管组 |
| [sparkx-edit-ai-group](skills/sparkx-edit-ai-group/) | `edit_sd_ai_managed_group` / `save_sp_sb_ai_managed_group` | 编辑单个或批量 AI 托管组 |
| [sparkx-delete-ai-group](skills/sparkx-delete-ai-group/) | `delete_ai_managed_group` | 删除托管组，并释放或迁移其中的 Campaign |

**可选（4 个）**——进阶分析场景，装完必装 Skills 后按需添加：

| Skill | 用途 |
|-------|------|
| [sparkx-weekly-ads-report](skills/sparkx-weekly-ads-report/) | 广告周报：KPI 环比、7 天趋势、异常摘要、Top 变化榜、下周行动建议 |
| [sparkx-monthly-ads-report](skills/sparkx-monthly-ads-report/) | 广告月报：全月 KPI（环比 + 同比）、结构拆解、商品与关键词分析 |
| [sparkx-ads-structure-analysis](skills/sparkx-ads-structure-analysis/) | 广告结构分析：按广告类型 / 站点 / 组合等维度定位结构错配 |
| [sparkx-product-diagnosis](skills/sparkx-product-diagnosis/) | 商品诊断：ASIN 健康度分层、变体对比、去留优化建议 |

**安装方式**（任选其一）：

- **Claude Code / Codex / Cursor 等**：把 [`skills/`](skills/) 目录发给 AI，说一句"帮我把 6 个必装 Skill 装上"，可选 Skills 按需加装。
- **Claude 网页版 / 桌面 App 等界面类助手**：设置 → Skills → 上传，逐个添加。

各 Skill 版本见 [skills/manifest.json](skills/manifest.json)，版本历史见 [CHANGELOG.md](CHANGELOG.md)。

---

## 示例提问

- 「昨天我账户下所有店铺表现如何？找出最需要关注的 5 个问题。」
- 「上周花费最高的 10 个 campaign，带 ACOS。」
- 「最近 30 天对比这几个产品线的表现，从广告结构和定向类型角度给优化建议。」
- 「过去 7 天有哪些 AI 自动调价？分别是为什么？」
- 「谁在什么时候改了这个 campaign 的预算？」
- 「把这 3 个 SP Campaign 创建为一个新的 AI 托管组，先展示完整配置让我确认。」
- 「把这个托管组的目标 ACOS 改为 25%，执行前列出原值和新值。」

**提示词技巧**：说清时间范围、维度、指标、排序、Top N；点名店铺；一次一个意图，复杂需求拆开问。

---

## 当前版本边界

- **写操作直接生效**：创建、编辑和删除托管组会直接修改线上配置。只应向可信用户授予写入或删除权限；执行前必须核对 Profile、对象和变更内容并取得明确确认，执行后必须回查。
- **写入范围**：当前仅支持 AI 托管组管理，不支持直接创建、编辑或删除 Campaign。
- **暂不支持**：托管组排期、通过模板设置托管组、词库相关设置；RBA 配置不可读取或修改。行动空间允许从 RBA 切换为 AI，但不支持从 AI 切换为 RBA。
- **范围**：你能查的店铺，与你的 SparkX 账号（主 / 子账号）权限一致。
- **历史回溯**：约可查最近 15 个月。
- **非秒级实时**：与 SparkX AI 平台数据更新节奏一致；效果数据最细到天。

---

## Plugin 安装指南

### 桌面端 App

> 普通 Claude Desktop / Claude 网页版的 Chat 模式请使用 [Custom Connector](docs/installation.md#claude-desktop-与-claude-网页版custom-connector无需配置文件) 连接 MCP。Custom Connector 只提供 MCP Tools，Skills 需要在 **Settings → Skills** 中单独添加。Claude Team / Enterprise 需要先由组织的 Owner 或 Primary Owner 添加连接器。以下 marketplace 安装流程仅适用于 Claude Code Desktop。

#### Claude Code Desktop

1. 切换到 **Code** 标签页。
2. 点击输入框旁的 **+ → Plugins → Add plugin**，打开 Plugin browser。
3. 在 **Marketplaces** 中选择从 repository 添加，输入 `SparkXAI/MCP`。
4. 找到并安装 **SparkX AI MCP**。Plugin 会同时安装 MCP server 和全部 10 个 Skills。

> Claude Code Desktop 不支持 `/plugin` 命令；`/plugin` 仅用于 Claude Code CLI。桌面端请使用上述 Plugin browser。

#### ChatGPT Desktop · Codex

1. 点击左侧边栏的 **Plugins**。
2. 点击右上角的 **Add → Add a marketplace**。
3. 输入 `SparkXAI/MCP` 并添加 marketplace。
4. 在 **Plugins** 中找到新添加的 marketplace，然后安装 **SparkX AI MCP**。
5. 新建一个 Codex 任务，MCP server 和全部 10 个 Skills 将在新任务中可用。

> ChatGPT 桌面端 Codex 不支持 `/plugin` 或 `/plugins` 命令；`/plugins` 仅用于 Codex CLI。桌面端必须通过左侧边栏的 Plugins 操作。

桌面端界面参考：[Claude Code Desktop](https://code.claude.com/docs/en/desktop#install-plugins) · [ChatGPT / Codex Plugins](https://learn.chatgpt.com/docs/plugins)

### 命令行 CLI

#### Claude Code CLI

```text
/plugin marketplace add SparkXAI/MCP
/plugin install sparkx-ai-mcp@sparkx-ai
```

#### Codex CLI

```bash
codex plugin marketplace add SparkXAI/MCP
codex
```

进入 Codex CLI 后输入 `/plugins`，从 `sparkx-ai` marketplace 中选择并安装 **SparkX AI MCP**，然后启动新会话。

Plugin 会同时安装 MCP server 和全部 10 个 Skills。当前 Plugin 配置使用 `SPARKX_AI_TOKEN`；如需使用 OAuth，请按[安装说明中的 OAuth 配置](docs/installation.md#oauth)连接。

Marketplace：[Claude Code](.claude-plugin/marketplace.json) · [Codex](.agents/plugins/marketplace.json)；Plugin descriptor：[Claude Code](.claude-plugin/plugin.json) · [Codex](.codex-plugin/plugin.json)

### 通用 MCP 客户端

参考[安装说明 · 方式三 · 手动配置](docs/installation.md#方式三--手动配置)，获取 Hermes、OpenClaw、Cherry Studio、Cursor、Cline 等客户端的配置示例。

---

## 安全提示

> OAuth 和 MCP Token 的可访问范围均受 SparkX 账号权限、授权范围和店铺范围限制。OAuth 授权不再使用时请及时撤销；MCP Token 请妥善保管、勿外传，并建议通过环境变量配置。发现异常访问时，应立即撤销授权或禁用 Token。
