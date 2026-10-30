# Blockchain Data Engineering Journey

记录我的 Blockchain Data Engineering 学习过程、实践项目和开源贡献进度。

## 学习与贡献目标

- 学习区块链数据结构、链上事件、交易与数据建模，提升 SQL 和 DuneSQL 能力。
- 掌握 Python、Git、uv 和 dbt 工作流，形成可复现的开发与排错记录。
- 通过 Dune Spellbook 理解区块链数据从原始数据到分析模型的转换过程。
- 从阅读模型、验证问题和完善文档开始，逐步参与模型改进、测试与 Pull Request 贡献。

## 项目记录

| 项目 | 当前进度 | 文档 |
| --- | --- | --- |
| Dune Spellbook | 核心模型与 seed test 基础阅读完成；下一课回到跨链 lineage；Dune 模型输出验证待开展 | [项目进度](projects/spellbook/README.md) · [环境搭建](projects/spellbook/01-environment-setup.md) · [DEX 模型分析](projects/spellbook/02-dex-model-analysis.md) · [单条 Swap 核对](projects/spellbook/2026-10-29-swap-seed-verification.md) · [测试与学习路径](projects/spellbook/2026-10-30-dbt-tests-and-learning-path.md) |

## 仓库边界与记录原则

这是独立的个人学习文档仓库，本地路径为 `C:\Dev\blockchain-data-engineering-journey`。Spellbook 开发目录为 `C:\Dev\spellbook`，两者独立管理。

学习记录应区分环境搭建、本地编译、数据执行与数据质量验证。`dbt compile` 成功仅代表本地编译阶段通过，不代表已查询真实链上数据，也不代表数据质量验证通过。

仓库仅保存学习文档与可公开的实践资料，不保存 API Key、凭据或包含敏感配置的 `profiles.yml`。
