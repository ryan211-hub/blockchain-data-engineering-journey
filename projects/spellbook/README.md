# Dune Spellbook 学习与贡献进度

通过 [Dune Spellbook](https://github.com/duneanalytics/spellbook) 学习 DuneSQL、dbt 模型组织和区块链数据工程的开源协作流程。

## 相关位置

- 个人文档仓库：`C:\Dev\blockchain-data-engineering-journey`
- Spellbook 开发目录：`C:\Dev\spellbook`
- [Notion 学习记录](https://app.notion.com/p/40619faa63d681f19336d53e4ae99ac5)
- [Windows 开发环境搭建记录](01-environment-setup.md)
- [DEX Swap 模型分析：Uniswap V3 到美元估值](02-dex-model-analysis.md)
- [Notion 模型阅读记录](https://app.notion.com/p/40719faa63d681a4bc4ce7e6be731017)

本目录保存个人学习进度，不将个人日志写入 Spellbook 上游项目或本地开发目录。

## 当前进度

环境搭建结果来自此前记录；2026-10-28 新增核心模型阅读记录。本次文档更新未重新运行 dbt 命令。

| 阶段 | 状态 | 结果 |
| --- | --- | --- |
| Windows 基础工具 | 已完成 | Python 3.13.5、Git 2.53.0、uv 0.13.6 |
| dbt 环境 | 已完成 | dbt-core 1.12.4、dbt-trino 1.10.3 |
| Codex 扩展版本调整 | 已完成 | 已降级到 26.1007.21434 |
| dbt profiles-dir 问题处理 | 已完成 | 处理配置目录定位问题后，后续命令成功 |
| dbt deps | 已完成 | 依赖安装成功，退出码 0 |
| dbt compile | 已完成 | 本地编译成功，退出码 0；manifest.json 存在 |
| DEX Swap 核心模型阅读 | 已完成 | 从 Uniswap V3 base 到 Ethereum 汇总、metadata 与美元估值；已读源码及已有编译 SQL，未执行数据 tests |
| 真实链上查询与数据质量验证 | 待开展 | 本地编译结果不能作为数据验证通过的证据 |
| 开源贡献 PR | 待开展 | 后续选定具体问题并按项目流程验证、提交 |

## 下一步

1. 在具备 Dune 查询条件后，选择一条真实 Swap 或短日期窗口，核对源事件、池信息、base 输出及美元金额。
2. 单独记录查询与数据质量验证证据；不预设模型存在 Bug。
3. 继续学习 `dex.trades` 跨链汇总与 tests 的详细实现。
4. 根据证据选择具体 Issue，完成必要验证后参与开源贡献。
