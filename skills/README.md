# SparkX MCP Skills

三个官方 Skill，配合 SparkX MCP 使用。**没有 Skills，MCP 查询的出错率会明显上升**——强烈建议安装。

| Skill | 版本 | 对应 MCP Tool | 所需 Scope | 用途 |
|-------|------|--------------|-----------|------|
| [query-ads-performance](query-ads-performance/) | 1.0.0 | `get_ads_perf` | `amazon_sa:performance:read` | 查询广告效果指标：花费、ACOS、ROAS、趋势、排名、同环比 |
| [query-entity-metadata](query-entity-metadata/) | 1.0.0 | `get_entity_metadata` | `amazon_sa:ads_configuration:read` | 查询实体配置：广告活动 / 广告组 / 投放 / ASIN / 托管组的名称、状态、设置 |
| [query-operation-log](query-operation-log/) | 1.0.0 | `get_operation_log` | `amazon_sa:ads_logs:read` | 查询操作日志：人工与 AI 的调价、调预算、启停记录 |

## 版本

- **程序读取**：[`manifest.json`](manifest.json) 是机器可读的版本清单（含各 skill 的 version、对应 tool、scope 和最新打包下载地址），可通过固定地址获取：

  ```
  https://raw.githubusercontent.com/SparkXAI/MCP/main/skills/manifest.json
  ```

- **人工查看**：每个 Skill 的版本号在其 `SKILL.md` frontmatter 的 `version` 字段；整体变更历史见 [CHANGELOG.md](../CHANGELOG.md)。
- 发布新版本时，`manifest.json` 与 frontmatter 会同步更新，并打对应的 git tag / GitHub Release。

## 安装

- **能让 AI 自己动手的助手**（Claude Code、ChatGPT Codex、Cursor 等）：把本目录（或 [Releases](../../../releases) 里的打包版）发给它，说一句"帮我把这三个 Skill 装上"。
- **聊天 / 界面类助手**（Claude 网页版或桌面 App、Cherry Studio、扣子 Coze 等）：在各自设置里手动添加（以 Claude 为例：设置 → Skills → 上传，逐个加上三个）。

## 目录结构

每个 Skill 是一个独立目录：

```
query-ads-performance/
├── SKILL.md          # Skill 主文件（含 name / version / description 与方法论）
└── references/       # 字段参考、枚举值、示例查询等
```
