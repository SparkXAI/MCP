# Changelog

本文件记录 SparkX AI MCP Skills 与配套文档的版本变化。

## [Unreleased]

### 变更

- **sparkx-query-entity-metadata** `1.1.0` → `1.1.1`。
- **sparkx-query-operation-log** `1.1.0` → `1.1.1`。
- **sparkx-create-ai-group** `1.0.0` → `1.0.1`：增加业务术语消歧、写入前强制澄清检查，并修正创建时预算与目标字段的映射。
- **sparkx-edit-ai-group** `1.0.0` → `1.0.1`：增加业务术语消歧，区分需求澄清与写入授权，并按调用路径处理状态枚举。
- **sparkx-query-entity-metadata** `1.1.1` → `1.1.2`：补充托管组总预算汇总口径及其按比例调整 Campaign 预算的行为。
- **sparkx-create-ai-group** `1.0.1` → `1.0.2`：明确托管组预算、按表现调预算增量上限、预算重新分配作用范围及创建确认要求。
- **sparkx-edit-ai-group** `1.0.1` → `1.0.2`：补充固定值/百分比、Campaign/整组预算作用模型，可靠查询启用 Campaign，并在写入前展示完整影响。
- **sparkx-delete-ai-group** `1.0.0` → `1.0.1`：不再依赖 Campaign `aiGroupId` 服务端过滤，改为完整分页后本地校验托管组归属。
- 增加 Skill 版本检查指引：AI Agent 在安装或更新时对比本地 `SKILL.md` 与远端 `manifest.json`，发现本地版本较低时提醒用户更新。
- 增加 OAuth 请求前校验要求：AI Agent 必须按 MCP 授权流程完成 discovery，并在发送请求前校验协议要求的全部必填字段。

## [1.1.0] - 2026-08-18

### 变更

- MCP 连接新增 OAuth 授权，与现有 MCP Token 两种方式并行支持。
- Plugin 内置配置继续使用 MCP Token；支持 OAuth 的客户端可通过 Custom Connector 或手动配置连接。
- 更新 README、安装说明、连接验证和 401 排障指引，明确 OAuth 与 MCP Token 两条配置路径。
- 新增 3 个 **1.0.0 必装 Skills**：`sparkx-create-ai-group`、`sparkx-edit-ai-group`、`sparkx-delete-ai-group`，覆盖 SP、SB、SD AI 托管组的创建、编辑和删除。
- 明确托管组写操作会直接修改线上配置，要求执行前确认、执行后回查。
- 补充当前能力边界：不支持排期、模板和词库设置；RBA 配置不可读取或修改，支持 RBA → AI，不支持 AI → RBA。

## [1.0.0] - 2026-07-28

首个版本（对应 MCP Server v1.0.0 · 数据查询）。

### 新增

- 3 个**必装 Skills**：
  - **sparkx-query-ads-performance** `1.0.0` — 广告效果查询 Skill，对应 MCP tool `get_ads_perf`（scope：`amazon_sa:performance:read`）
  - **sparkx-query-entity-metadata** `1.0.0` — 实体配置 / 元数据查询 Skill，对应 MCP tool `get_entity_metadata`（scope：`amazon_sa:ads_configuration:read`）
  - **sparkx-query-operation-log** `1.0.0` — 操作日志查询 Skill，对应 MCP tool `get_operation_log`（scope：`amazon_sa:ads_logs:read`）
- 4 个**可选 Skills**（位于 `skills/optional/`，装完必装 Skills 后按需添加）：
  - **sparkx-weekly-ads-report** `0.9.4` — 广告周报
  - **sparkx-monthly-ads-report** `0.5.6` — 广告月报
  - **sparkx-ads-structure-analysis** `0.2.3` — 广告结构分析
  - **sparkx-product-diagnosis** `0.1.11` — 商品诊断
- `skills/manifest.json` 机器可读版本清单（含 `required` 字段区分必装 / 可选）
- README 与安装说明（Claude / ChatGPT Codex / WorkBuddy / Cherry Studio / 扣子 Coze / OpenClaw / Hermes / Cursor 等客户端配置）
