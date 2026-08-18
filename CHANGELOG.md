# Changelog

本文件记录 SparkX AI MCP Skills 与配套文档的版本变化。

## [Unreleased]

## [1.1.0] - 2026-08-18

### 变更

- MCP 连接新增 OAuth 授权，与现有 MCP Token 两种方式并行支持。
- Plugin 内置配置继续使用 MCP Token；支持 OAuth 的客户端可通过 Custom Connector 或手动配置连接。
- 更新 README、安装说明、连接验证和 401 排障指引，明确 OAuth 与 MCP Token 两条配置路径。
- 新增 3 个 **1.0.0 必装 Skills**：`create-ai-group`、`edit-ai-group`、`delete-ai-group`，覆盖 SP、SB、SD AI 托管组的创建、编辑和删除。
- 明确托管组写操作会直接修改线上配置，要求执行前确认、执行后回查。
- 补充当前能力边界：不支持排期、模板和词库设置；RBA 配置不可读取或修改，支持 RBA → AI，不支持 AI → RBA。

## [1.0.0] - 2026-07-28

首个版本（对应 MCP Server v1.0.0 · 数据查询）。

### 新增

- 3 个**必装 Skills**：
  - **query-ads-performance** `1.0.0` — 广告效果查询 Skill，对应 MCP tool `get_ads_perf`（scope：`amazon_sa:performance:read`）
  - **query-entity-metadata** `1.0.0` — 实体配置 / 元数据查询 Skill，对应 MCP tool `get_entity_metadata`（scope：`amazon_sa:ads_configuration:read`）
  - **query-operation-log** `1.0.0` — 操作日志查询 Skill，对应 MCP tool `get_operation_log`（scope：`amazon_sa:ads_logs:read`）
- 4 个**可选 Skills**（位于 `skills/optional/`，装完必装 Skills 后按需添加）：
  - **weekly-ads-report** `0.9.4` — 广告周报
  - **monthly-ads-report** `0.5.6` — 广告月报
  - **ads-structure-analysis** `0.2.3` — 广告结构分析
  - **product-diagnosis** `0.1.11` — 商品诊断
- `skills/manifest.json` 机器可读版本清单（含 `required` 字段区分必装 / 可选）
- README 与安装说明（Claude / ChatGPT Codex / WorkBuddy / Cherry Studio / 扣子 Coze / OpenClaw / Hermes / Cursor 等客户端配置）
