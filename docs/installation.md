# SparkX MCP 安装说明

完整走完 4 步：**获取 token → 配置 MCP → 安装 Skills → 验证**。全程约 5 分钟。

> 前提：你需要一个**支持 MCP 的 AI 助手**。最简单、最推荐用 **WorkBuddy**（国内）或 **Claude**（海外，Desktop 应用或 Code CLI）；也支持 ChatGPT Codex、Cherry Studio、扣子 Coze、OpenClaw、Hermes、Cursor、Cline 等。

---

## 第一步 · 获取你的 token

> ⚠️ Token 决定你能查询哪些店铺和哪些数据，请妥善保管、勿外传。

1. 登录 SparkX 后台，点击右上角账号菜单，进入 **MCP & Skills**。
2. 点击「**新建 Token**」，按需选择：有效期、授权范围（如 Amazon SA）、店铺范围和数据权限。
3. **立即复制并妥善保存** 生成的 token——关闭弹窗后将无法再次查看，遗失需重新创建。

---

## 第二步 · 在你的 AI 助手里配置 MCP

**本质就一件事**：告诉助手两样东西——Server URL 和你的 token（作为 `Authorization: Bearer <token>` 请求头）。

- **Server URL**：`https://mcp.sparkx.cn/mcp`
- **传输方式**：Streamable HTTP

### 方式一 · 一句话配置（推荐）

在能让 AI 自己动手配置的客户端（Claude、ChatGPT Codex、Cherry Studio、扣子 Coze、WorkBuddy 等）里，直接把下面这段发给它：

```text
帮我配置 SparkX MCP
URL：https://mcp.sparkx.cn/mcp
Token：<你的 token>
配置完调 get_user_authorized_context 验证。
```

它会自动配好并验证，返回你授权的店铺列表就说明成功。

- **Claude**：打开 Claude → 切到 **Code 标签页** → 粘贴上面这段。
- **ChatGPT Codex**：直接把这段发给 Codex（它会写进 `~/.codex/config.toml`）。
- **Cherry Studio / 扣子 Coze / WorkBuddy**：在对话里发给对应智能体即可。

若客户端不支持让 AI 代配，按下方「手动配置」在设置里添加。

### 方式二 · 手动配置

#### Claude Code CLI / Desktop 的 Code 标签页

```bash
claude mcp add --transport http sparkx-mcp https://mcp.sparkx.cn/mcp --header "Authorization: Bearer <你的TOKEN>"
```

建议保持默认的 **local scope**（仅当前项目文件夹），token 不会散落到其他项目。

#### Claude Desktop（Chat，UI 无 Bearer 选项时）

编辑 `claude_desktop_config.json`（Windows：`%APPDATA%\Claude\`；macOS：`~/Library/Application Support/Claude/`），用 `mcp-remote` 包一层后重启：

```json
{
  "mcpServers": {
    "sparkx-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://mcp.sparkx.cn/mcp", "--header", "Authorization: Bearer ${SPARKX_TOKEN}"],
      "env": { "SPARKX_TOKEN": "<你的token>" }
    }
  }
}
```

#### ChatGPT Codex（`~/.codex/config.toml`）

```toml
[mcp_servers.sparkx-mcp]
url = "https://mcp.sparkx.cn/mcp"
bearer_token_env_var = "SPARKX_TOKEN"
http_headers = {}
```

#### OpenClaw

```bash
openclaw mcp add sparkx-mcp \
  --url https://mcp.sparkx.cn/mcp \
  --transport streamable-http \
  --header "Authorization: Bearer $SPARKX_TOKEN"
```

> ⚠️ 旧版 OpenClaw 有个 bug：streamable-http 不转发自定义 Authorization 头（约 2026 年 4 月底起已修复）。遇到 401 先升级到最新版。

#### Hermes（`~/.hermes/config.yaml`）

```yaml
mcp_servers:
  sparkx-mcp:
    url: "https://mcp.sparkx.cn/mcp"
    headers:
      Authorization: "Bearer ${SPARKX_TOKEN}"
```

然后在 Hermes 里运行 `/reload-mcp`。

#### 其他 MCP 客户端（Cursor、Cline 等）

在客户端的 MCP 设置里添加一个远程 / Streamable HTTP server，填上面的 URL 和 `Authorization: Bearer <你的TOKEN>` 请求头即可。

### 验证连通性

让助手调用 `get_user_authorized_context`：

| 结果 | 含义 |
|------|------|
| 返回你的 userId 和授权的 profileIds | ✅ 配置成功 |
| 返回 401 | Token 错误或缺权限，重新生成 |
| 超时 | 检查网络 |

---

## 第三步 · 安装 Skills

Skills 在本仓库 [`skills/`](../skills/) 目录，分两类：

**必装（3 个）**——没有它们，MCP 查询的出错率会明显上升：

- `query-ads-performance` — 广告效果查询
- `query-entity-metadata` — 实体配置查询
- `query-operation-log` — 操作日志查询

**可选（4 个）**——进阶分析场景，装完必装的再按需添加（位于 [`skills/optional/`](../skills/optional/)）：

- `weekly-ads-report` — 广告周报
- `monthly-ads-report` — 广告月报
- `ads-structure-analysis` — 广告结构分析
- `product-diagnosis` — 商品诊断

按你的助手选安装方式：

- **能让 AI 自己动手的助手**（Claude Code、ChatGPT Codex、Cursor 等）：把 Skill 目录发给它，说一句"帮我把必装的三个 Skill 装上"，可选 Skills 按需加装。
- **聊天 / 界面类助手**（Claude 网页版或桌面 App、Cherry Studio、扣子 Coze 等）：在各自设置里手动添加（以 Claude 为例：设置 → Skills → 上传，逐个添加）。

---

## 第四步 · 开始查询

四步走：1）让助手列出你授权的店铺 → 2）选一个店铺 → 3）用自然语言提问 → 4）拿走结果（让它导出表格 / 图表 / 放进报告）。

**示例提示词：**

- 表现：
  - "上周花费最高的 10 个 campaign，带 ACOS。"
  - "这个产品线过去 8 周的 TACOS 趋势。"
  - "本周相比上周 ACOS 上升最多的 5 个 campaign。"
- 配置：
  - "列出所有启用的 SP campaign 及其日预算。"
  - "这个 ASIN 的库存、标题和广告类型资格。"
- 操作日志：
  - "过去 7 天 AI 自动调价的记录。"
  - "谁在什么时候改了这个 campaign 的预算？"

**进阶**：让助手把多个查询综合成一份周报、导出 CSV/Excel，或结合你的成本/目标算出真实利润；把常用提示词做成可复用模板。

---

## 常见问题

**Q：返回 401 Unauthorized？**
Token 过期、复制不完整或权限不足。到后台 MCP & Skills 页面重新生成一个，注意勾选对应店铺和数据权限。旧版 OpenClaw 用户请先升级客户端。

**Q：能查多久以前的数据？**
效果数据和操作日志约可回溯最近 15 个月，效果数据最细到天。

**Q：数据是实时的吗？**
与 SparkX AI 平台数据更新节奏一致，非秒级实时。

**Q：能查哪些店铺？**
与你的 SparkX 账号（主 / 子账号）权限一致——token 的可见范围不会超过你本人的账号权限。

**Q：这一版能改预算 / 调竞价吗？**
不能。v1.0.0 是只读版本，写操作能力（托管组管理、批量操作、活动创建）将在后续版本陆续开放。
