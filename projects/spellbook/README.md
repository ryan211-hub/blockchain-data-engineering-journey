# Dune Spellbook 学习与贡献进度

通过 [Dune Spellbook](https://github.com/duneanalytics/spellbook) 学习 DuneSQL、dbt 模型组织和区块链数据工程的开源协作流程。

## 相关位置

- 个人文档仓库：`C:\Dev\blockchain-data-engineering-journey`
- Spellbook 开发目录：`C:\Dev\spellbook`
- [Notion 学习记录](https://app.notion.com/p/40619faa63d681f19336d53e4ae99ac5)
- [Windows 开发环境搭建记录](01-environment-setup.md)

本目录保存个人学习进度。此次文档初始化不修改 Spellbook 上游项目或本地开发目录。

## 当前进度

以下已完成事项按本次提供的环境搭建结果整理；本次文档初始化未重新运行 dbt 命令。

| 阶段 | 状态 | 结果 |
| --- | --- | --- |
| Windows 基础工具 | 已完成 | Python 3.13.5、Git 2.53.0、uv 0.13.6 |
| dbt 环境 | 已完成 | dbt-core 1.12.4、dbt-trino 1.10.3 |
| Codex 扩展版本调整 | 已完成 | 已降级到 26.1007.21434 |
| dbt profiles-dir 问题处理 | 已完成 | 处理配置目录定位问题后，后续命令成功 |
| dbt deps | 已完成 | 依赖安装成功，退出码 0 |
| dbt compile | 已完成 | 本地编译成功，退出码 0；manifest.json 存在 |
| 真实链上查询与数据质量验证 | 待开展 | 本地编译结果不能作为数据验证通过的证据 |
| 开源贡献 PR | 待开展 | 后续选定具体问题并按项目流程验证、提交 |

## 下一步

1. 阅读 Spellbook 的仓库结构、模型规范和测试要求，选择一个具体模型进行学习。
2. 理解模型的 `source()`、`ref()`、增量逻辑与测试配置，检查编译 SQL。
3. 在具备查询条件后执行 Dune 查询并核对结果，单独记录数据验证证据。
4. 完成必要测试后，整理可审查的改动并参与开源贡献。
