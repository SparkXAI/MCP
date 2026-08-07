# SparkX AI MCP Skills

配合 SparkX AI MCP 使用的官方 Skills，分两类：

- **必装（3 个）**：基础查询能力。**没有它们，MCP 查询的出错率会明显上升**，请务必安装。
- **可选（4 个）**：面向具体分析场景的进阶玩法，按需安装。

## 必装 Skills

| Skill | 版本 | 对应 MCP Tool | 所需 Scope | 用途 |
|-------|------|--------------|-----------|------|
| [query-ads-performance](query-ads-performance/) | 1.0.0 | `get_ads_perf` | `amazon_sa:performance:read` | 查询广告效果指标：花费、ACOS、ROAS、趋势、排名、同环比 |
| [query-entity-metadata](query-entity-metadata/) | 1.0.0 | `get_entity_metadata` | `amazon_sa:ads_configuration:read` | 查询实体配置：广告活动 / 广告组 / 投放 / ASIN / 托管组的名称、状态、设置 |
| [query-operation-log](query-operation-log/) | 1.0.0 | `get_operation_log` | `amazon_sa:ads_logs:read` | 查询操作日志：人工与 AI 的调价、调预算、启停记录 |

## 可选 Skills

基于上面三个必装 Skill 的查询能力做进阶分析，装了必装 Skills 之后即可按需添加：

| Skill | 版本 | 用途 |
|-------|------|------|
| [weekly-ads-report](weekly-ads-report/) | 1.0.0 | 广告周报：KPI 环比卡片、7 天趋势、异常摘要、Top 变化榜、下周行动建议 |
| [monthly-ads-report](monthly-ads-report/) | 1.0.0 | 广告月报：全月 KPI（环比 + 同比）、结构拆解、商品与关键词分析、下月建议 |
| [ads-structure-analysis](ads-structure-analysis/) | 1.0.0 | 广告结构分析：按广告类型 / 站点 / 组合 / 工作日维度拆解花费与效率，定位结构错配 |
| [product-diagnosis](product-diagnosis/) | 1.0.0 | 商品诊断：ASIN 健康度分层、变体对比、问题商品诊断卡、去留优化建议 |

## 版本

- **程序读取**：[`manifest.json`](manifest.json) 是机器可读的版本清单（含各 skill 的 version、必装/可选标记、对应 tool 和 scope），可通过固定地址获取：

  ```
  https://raw.githubusercontent.com/SparkXAI/MCP/main/skills/manifest.json
  ```

- **人工查看**：每个 Skill 的版本号在其 `SKILL.md` frontmatter 的 `version` 字段；整体变更历史见 [CHANGELOG.md](../CHANGELOG.md)。
- 发布新版本时，`manifest.json` 与 frontmatter 会同步更新，并打对应的 git tag / GitHub Release。

## 安装

- **能让 AI 自己动手的助手**（Claude Code、ChatGPT Codex、Cursor 等）：把 Skill 目录发给它，说一句"帮我装上"。先装必装的三个，再按需加可选的。
- **聊天 / 界面类助手**（Claude 网页版或桌面 App、Cherry Studio、扣子 Coze 等）：在各自设置里手动添加（以 Claude 为例：设置 → Skills → 上传，逐个添加）。

## 目录结构

```
skills/
├── manifest.json              # 机器可读版本清单
├── query-ads-performance/     # 必装
├── query-entity-metadata/     # 必装
├── query-operation-log/       # 必装
├── weekly-ads-report/         # 可选
├── monthly-ads-report/        # 可选
├── ads-structure-analysis/    # 可选
└── product-diagnosis/         # 可选
```

每个 Skill 是一个独立目录，含 `SKILL.md`（name / version / description 与方法论）和 `references/`（字段参考、示例查询等）。
