# Changelog

本文件记录 SparkX AI MCP Skills 与配套文档的版本变化。

## [Unreleased]

### 变更

- MCP 连接新增 OAuth 授权，并将 OAuth 设为 Plugin 默认方式。
- 保留 MCP Token，供暂不支持 OAuth 的客户端、自动化脚本和固定凭证场景使用。
- 更新 README、安装说明、连接验证和 401 排障指引，明确 OAuth 与 MCP Token 两条配置路径。

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
