# MVP 竞品更新自动监控工作流方案

## Summary

搭建一个无自建服务的 MVP：GitHub Actions 定时并行触发 Dify Workflow，Dify 负责官网发布日志抓取、AI 结构化分析、飞书多维表格查重入库，并在有新增时通过飞书群机器人推送表格链接；每周五 16:00 生成 Markdown 周报推送到飞书群。

默认监控对象：Cursor、Claude Code、Codex、GitHub Copilot。P0 数据源只做官网/官方发布日志，P1 的 X、B站、YouTube 暂不纳入。

## Key Changes

- 建两个 Dify Workflow：
  - `daily_competitor_scan`：输入单个竞品配置，抓取发布日志，抽取新增更新，查询飞书去重，写入新增记录，若有新增则推送飞书表格链接。
  - `weekly_competitor_report`：查询本周飞书更新记录，生成“重点更新 + 分类汇总 + 建议动作”的 Markdown 周报，并推送飞书机器人。
- GitHub Actions 负责定时和并行：
  - 每天北京时间 `09:00`、`22:00` 触发日常扫描，对每个竞品使用 matrix 并行调用 Dify。
  - 每周五北京时间 `16:00` 触发周报 Workflow。
  - UTC cron：`0 1,14 * * *` 日常扫描，`0 8 * * 5` 周报。
- GitHub repo 配置文件维护：
  - 竞品名称、官网发布日志 URL、抓取规则提示、是否启用。
  - AI 分析 Prompt、更新类型标签、字段输出 schema。
  - 默认更新类型：`功能`、`模型`、`定价`、`生态`、`集成`、`营销`、`其他`。
- P0 官方 URL 初版配置：
  - Cursor：[https://cursor.com/changelog](https://cursor.com/changelog)
  - Claude Code：[https://code.claude.com/docs/en/changelog](https://code.claude.com/docs/en/changelog)
  - Codex：[https://help.openai.com/en/articles/11428266-codex-changelog/](https://help.openai.com/en/articles/11428266-codex-changelog/)
  - GitHub Copilot：优先使用 GitHub 官方 Changelog 的 Copilot 分类/搜索入口；GitHub Docs 明确建议跟踪 Copilot changelog。
- 飞书多维表格建两张表：
  - `竞品更新表`：一条更新一行。
  - `运行日志表`：记录每次竞品扫描的成功、失败、无更新、错误信息和运行时间。
- `竞品更新表`字段：
  - `去重键`：由 `竞品 + 原文链接 + 标题 + 发布日期` 生成。
  - `竞品名称`
  - `更新标题`
  - `发布日期`
  - `抓取时间`
  - `50字摘要`
  - `更新类型`：多选。
  - `功能详情`、`模型详情`、`定价详情`、`生态详情`、`集成详情`、`营销详情`、`其他详情`：每列 50 字以内。
  - `详细描述`：200 字以内。
  - `原文链接`
  - `对 zcode 的产品影响判断`
  - `建议动作`
  - `人工重要性`：单选，默认 `待评估`，可选 `高`、`中`、`低`、`待评估`。
- zcode 分析参照：
  - zcode 是“集成多家 agent 和模型的 AI 编程 IDE”。
  - 首要用户是个人开发者。
  - 当前阶段是 MVP/早期验证。
  - AI 不自动判断重要性，只输出产品影响和建议动作；重要性由人工在飞书维护。
- 飞书鉴权：
  - 使用自建飞书应用获取 `tenant_access_token`。
  - Dify 变量保存 `FEISHU_APP_ID`、`FEISHU_APP_SECRET`、`BITABLE_APP_TOKEN`、表 ID、机器人 Webhook。
  - GitHub Secrets 只保存 Dify Workflow API Key 和 Workflow Endpoint。

## Dify Workflow Tutorial Outline

- `daily_competitor_scan` 输入：
  - `competitor_name`
  - `source_url`
  - `type_labels`
  - `analysis_prompt_config`
  - `bitable_app_token`
  - `update_table_id`
  - `run_log_table_id`
- 日常扫描节点顺序：
  - HTTP 请求抓取 `source_url`。
  - LLM 节点从页面内容中提取更新条目，并输出 JSON。
  - Code/Template 节点生成每条记录的去重键。
  - HTTP 请求飞书多维表格，按去重键查询是否存在。
  - 只对不存在的记录调用飞书 API 写入。
  - 写入运行日志表。
  - 如果新增数量大于 0，调用飞书机器人 Webhook，只推送“本次发现新增，查看多维表格链接”。
  - 如果无新增，不推送。
  - 如果抓取失败，写运行日志，不推送。
- `weekly_competitor_report` 节点顺序：
  - HTTP 请求查询本周 `竞品更新表`记录。
  - LLM 生成 Markdown 周报。
  - 调用飞书机器人 Webhook 推送周报。
- LLM 配置：
  - 使用 Dify 中已接入的国内模型。
  - temperature 建议 `0.2`。
  - 强制 JSON 输出；解析失败时走一次修复 JSON 的重试节点。

## Test Plan

- 手动触发 GitHub Actions daily workflow，确认 4 个竞品会并行调用 Dify。
- 用一个测试 URL 或缩短后的页面内容验证：
  - 首次运行会写入飞书。
  - 第二次运行相同内容不会重复写入。
  - 无新增时不推送机器人。
  - 抓取失败时只写运行日志。
- 手动触发 weekly workflow，确认：
  - 能读取本周飞书记录。
  - 周报包含重点更新、按竞品汇总、按类型汇总、对 zcode 的影响和建议动作。
  - 飞书机器人 Markdown 可正常展示。
- 检查飞书表字段：
  - 多选类型能正常写入。
  - 每类详情列为空或 50 字以内。
  - `人工重要性`默认为 `待评估`。
- 检查 cron：
  - 日常扫描对应北京时间 09:00、22:00。
  - 周报对应北京时间周五 16:00。

## Assumptions

- MVP 不部署任何 FastAPI、Node、Serverless 或数据库服务。
- Dify HTTP 请求可以直接访问这些官网发布日志页面；如果某页面变成强动态渲染或反爬，MVP 只记录失败，不引入外部抓取服务。
- GitHub Actions repo 中会维护竞品配置和 Prompt/schema 配置。
- 日常推送只发飞书表格链接，不逐条推送详细内容。
- AI 分析内容后续通过 GitHub 配置文件调整，不需要每次进入 Dify 改 Prompt。
- P1 社媒源 X、B站、YouTube 暂不设计进 MVP，只保留未来扩展入口。
