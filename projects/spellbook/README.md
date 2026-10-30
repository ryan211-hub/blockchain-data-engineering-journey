# Dune Spellbook 学习与贡献进度

通过 [Dune Spellbook](https://github.com/duneanalytics/spellbook) 学习 DuneSQL、dbt 模型组织和区块链数据工程的开源协作流程。

## 相关位置

- 个人文档仓库：`C:\Dev\blockchain-data-engineering-journey`
- Spellbook 开发目录：`C:\Dev\spellbook`
- [Notion 学习记录](https://app.notion.com/p/40619faa63d681f19336d53e4ae99ac5)
- [Windows 开发环境搭建记录](01-environment-setup.md)
- [DEX Swap 模型分析：Uniswap V3 到美元估值](02-dex-model-analysis.md)
- [Notion 模型阅读记录](https://app.notion.com/p/40719faa63d681a4bc4ce7e6be731017)
- [2026-10-29：Uniswap V3 seed 与链上 Swap 人工核对](2026-10-29-swap-seed-verification.md)
- [Notion 单条 Swap 核对记录](https://app.notion.com/p/40819faa63d68158b992c1484d40ff30)
- [2026-10-30：dbt seed tests 与学习路径复核](2026-10-30-dbt-tests-and-learning-path.md)
- [Notion 测试与学习路径记录](https://app.notion.com/p/40919faa63d681b2ad02d58a747e2535)

本目录保存个人学习进度，不将个人日志写入 Spellbook 上游项目或本地开发目录。

## 当前进度

环境搭建结果来自此前记录；2026-10-28 完成核心模型阅读，2026-10-29 完成一条 seed 的链上人工核对，2026-10-30 完成 seed test 基础源码阅读并复核学习路径。本次文档更新未重新运行 dbt 命令。

| 阶段 | 状态 | 结果 |
| --- | --- | --- |
| Windows 基础工具 | 已完成 | Python 3.13.5、Git 2.53.0、uv 0.13.6 |
| dbt 环境 | 已完成 | dbt-core 1.12.4、dbt-trino 1.10.3 |
| Codex 扩展版本调整 | 已完成 | 已降级到 26.1007.21434 |
| dbt profiles-dir 问题处理 | 已完成 | 处理配置目录定位问题后，后续命令成功 |
| dbt deps | 已完成 | 依赖安装成功，退出码 0 |
| dbt compile | 已完成 | 本地编译成功，退出码 0；manifest.json 存在 |
| DEX Swap 核心模型阅读 | 已完成 | 从 Uniswap V3 base 到 Ethereum 汇总、metadata 与美元估值；已读源码及已有编译 SQL，未执行数据 tests |
| 单条 seed 与链上字段人工核对 | 已完成 | 学习者提供 Etherscan 事件与合约读取结果；买卖地址、raw amount、区块号及事件定位与 seed 一致，并换算数量 |
| Seed test 基础源码阅读 | 已完成 | 理解 YAML 绑定、匹配计数、字段比较、失败报告、唯一性与覆盖边界；仅阅读，未执行 tests |
| Cross-chain lineage | 下一课 | Ethereum 的交易记录怎样进入跨链 dex.trades；一次只看一个模型 |
| Dune 查询与模型数据质量验证 | 待开展 | 未读取 Dune 模型实际输出，未执行 dbt tests；人工核对与本地编译不等于模型数据验证通过 |
| 开源贡献 PR | 待开展 | 后续选定具体问题并按项目流程验证、提交 |

## 下一步

1. 测试专题基础阅读告一段落，不继续扩展假设性诊断 SQL 或直接决定补充 seed。
2. 下一课回到 `dex.trades` 跨链汇总：Ethereum 的交易记录怎样进入跨链模型，明确文件、model 与 database relation。
3. 架构主线读完后学习 contribution readiness：筛选适合首次贡献的任务，阅读 Issue 与 CI；根据证据选择具体问题。
4. 真实数据验证保持待办：具备 Dune 查询条件后核对源表、模型输出及美元金额；不预设 Bug，不把静态阅读标记为验证通过。
5. 按具体文件和业务问题小步推进；中文为主，约 30% 技术英文，跳过熟悉的基础 SQL。
