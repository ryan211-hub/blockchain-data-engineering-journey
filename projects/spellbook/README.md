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
- [2026-11-02：跨链交易接入与主动回忆学习](2026-11-02-cross-chain-integration.md)
- [Notion 跨链接入记录](https://app.notion.com/p/40c19faa63d68192b25cdc77c72ac327)
- [2026-11-02：增量窗口与历史回填小课题收束](2026-11-02-incremental-window-and-backfill.md)
- [Notion 增量窗口与回填记录](https://app.notion.com/p/40c19faa63d68129afd7c624d08ae73b)
- [2026-11-03：incremental_predicate 源码与两处调用](2026-11-03-incremental-predicate.md)
- [Notion 增量宏源码记录](https://app.notion.com/p/40d19faa63d681a7b885d68c6ced9361)

本目录保存个人学习进度，不将个人日志写入 Spellbook 上游项目或本地开发目录。

## 当前进度

环境搭建结果来自此前记录；2026-10-28 完成核心模型阅读，2026-10-29 完成一条 seed 的链上人工核对，2026-10-30 完成 seed test 基础源码阅读并复核学习路径，2026-11-02 完成跨链接入基础学习并调整为主动回忆式问答。本次文档更新未重新运行 dbt 命令。

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
| Cross-chain integration 基础 | 已学习并问答核对 | 链名单 → ref() → 各链模型 → dex.trades；字段接口、接入与质量区别、blockchain 匹配范围；不代表长期记忆或真实数据验证通过 |
| Incremental window / backfill 概念讨论 | 已收束 | 区分源读取窗口、目标匹配条件、独立回填与临时改窗口；通用数据处理原理不再反复测试 |
| Spellbook 增量实现 | 本课源码阅读完成 | 宏表达式、DEX 默认 3 / day 参数来源、源筛选与 MERGE ON 目标谓词、CI 单独分支；未核查运行参数，未编译或执行 |
| Dune 查询与模型数据质量验证 | 待开展 | 未读取 Dune 模型实际输出，未执行 dbt tests；人工核对与本地编译不等于模型数据验证通过 |
| 开源贡献 PR | 待开展 | 后续选定具体问题并按项目流程验证、提交 |

## 下一步

1. 测试专题基础阅读告一段落，不继续扩展假设性诊断 SQL 或直接决定补充 seed。
2. `incremental_predicate()` 小课题先总结同步；完成本地审核、提交推送与 Notion 发布状态更新后，再按既有 contribution readiness 方向明确下一小课题，不自动扩展回填操作或其他 merge 机制。
3. 架构主线读完后学习 contribution readiness：筛选适合首次贡献的任务，阅读 Issue 与 CI；根据证据选择具体问题。
4. 真实数据验证保持待办：具备 Dune 查询条件后核对源表、模型输出及美元金额；不预设 Bug，不把静态阅读标记为验证通过。
5. 聚焦项目特有实现，不反复测试熟悉的 SQL、数据关系、时间窗口、merge 与 backfill 原理。必要问题必须交代完整前提，不为互动而提问；中文为主，保留必要技术英文。
6. 固定课题完成流程：总结已学与待查事项 → 保存 Notion、展示 GitHub 本地 diff → 用户确认后 commit / push → 更新发布状态 → 再进入新知识点。
