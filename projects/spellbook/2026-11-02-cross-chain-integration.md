# 2026-11-02｜跨链交易接入与主动回忆学习

## 1. Learning checkpoint｜本次停在哪里

Cross-chain integration：已理解各链交易如何通过链名单和 ref() 进入 dex.trades，以及接入需要满足的字段接口。

Matching key：已理解不同链的记录应分别保留，blockchain 是跨链匹配范围的一部分；没有逐项深入 project / version 的业务场景。

Partitioning：仅初步理解 block_month 来自交易发生月份，不是模型运行月份。

Incremental processing：已出现“只读取近期交易，能否发现历史修正”的新问题，但尚未回答或展开。本节在这里暂停，不标记为已掌握。

## 2. Source map｜真实文件与对象

路径相对于 C:\Dev\spellbook。静态源码快照 HEAD：49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d；未重新运行编译或查询。

| SQL file / Macro | dbt model / Role | Database relation |
| --- | --- | --- |
| `dbt_subprojects/dex/models/trades/ethereum/dex_ethereum_trades.sql` | dex_ethereum_trades：Ethereum enrichment 后的交易 | dex_ethereum.trades |
| `dbt_subprojects/dex/models/trades/dex_trades.sql` | dex_trades：汇总各链标准化交易 | dex.trades |
| `dbt_subprojects/dex/macros/dex_evm_chains.sql` | dex_evm_chains()：提供链名称列表 | 宏，不是数据库表 |
| `dbt_subprojects/dex/macros/models/_project/uniswap_compatible_trades.sql` | 交易转换宏；含本次回顾的 block_month 表达式 | 宏，不是数据库表 |

Published relation 是正常发布语境下的名称；CI 可能构建到临时位置。本次没有访问生产表或 CI 表。

## 3. Cross-chain aggregation｜核心业务理解

**dex_trades 从各链模型读取交易数据，汇总成 dex.trades。**

Cross-chain aggregation means a unified dataset, not a cross-chain transfer，也不是把不同链的交易合并成同一笔交易。

入口先读取链列表：

```text
{% set chains = dex_evm_chains() %}
```

随后在循环中引用各链模型：

```text
{{ ref('dex_'~chain~'_trades') }}
```

| chain value | Generated dbt model name |
| --- | --- |
| ethereum | dex_ethereum_trades |
| arbitrum | dex_arbitrum_trades |
| example（练习用虚构链） | dex_example_trades |

链名单决定“读谁”，对应模型负责“提供数据”。添加 example 到名单只是练习，没有修改任何上游文件，也不是完整的新链接入流程。

入口用 UNION ALL 汇总各链结果，直接读取 amount_usd 等已处理字段。Enrichment happens upstream; union happens here。

## 4. Interface contract｜接入不仅需要模型名

这三项需要配套：

1. Chain list：链名被列入 dex_evm_chains()。
2. dbt model：存在 ref() 可以解析的对应模型。
3. Output contract：模型提供入口读取的字段，且类型兼容。

例如入口明确读取 amount_usd。如果上游根本没有该字段，保持当前入口不变就不能直接读取；字段存在但值为 NULL 则是另一回事。

```sql
CAST(NULL AS DOUBLE) AS amount_usd
```

这是学习时的占位方案示例，不是已实施修改，也不保证其余接口都已满足。

Missing column is not a NULL value。后续补算美元金额需要另外设计；当前跨链入口不会自动补齐估值。

## 5. Integration vs data quality｜接入成功不等于质量通过

如果 amount_usd 全为 NULL，只能确定美元金额不可用，不能声称估值已验证，也不能直接断定 enrichment 有 Bug。

Possible explanations include：

- Placeholder values：上游暂时输出 NULL 占位。
- Missing price data：没有可用的价格输入。
- Enrichment issue：关联或计算逻辑可能异常，需要证据确认。

Integration checks whether records can enter the dataset；data-quality validation checks whether values are available and correct。

本次没有检查任何真实 amount_usd 数据。对话中的全 NULL、新链 example、跨链相同哈希等都是 hypothetical examples。

## 6. Config and matching key｜配置与记录身份

以下是 dex_trades.sql 的实际配置节选：

```text
schema = 'dex'
, alias = 'trades'
, materialized = 'incremental'
, incremental_strategy = 'merge'
, partition_by = ['block_month', 'blockchain', 'project']
, unique_key = ['blockchain', 'project', 'version', 'tx_hash', 'evt_index', 'block_month']
```

Model name 是 dex_trades；schema + alias 对应正常发布的 dex.trades。配置中的 incremental / merge 已介绍基本含义，但执行窗口与历史修正留待下一主题。

unique_key 用于增量 merge 匹配，不是数据库自动创建的唯一约束。

练习：Ethereum 与 Arbitrum 的 tx_hash、evt_index 即使相同，也应作为不同链的记录保留。The event identity needs chain context。

分区字段也包含在 unique_key 中，这是当前项目的模型规范。业务事件身份与物理分区字段并非完全同一概念。

## 7. Partition boundary｜仅记录初步了解，不扩展增量主题

本次回顾的上游表达式：

```sql
CAST(date_trunc('month', dexs.block_time) AS date) AS block_month
```

Example：

```text
block_time  = 2022-09-28 14:35:00
block_month = 2022-09-01
```

2026 年 11 月重新处理这条历史交易，所属月份仍为 2022-09-01。Event time is not processing time。

这是 block_month 的初步认识，不代表已系统学习分区、增量窗口、历史回填或 merge 执行细节。

## 8. Learning method｜本次有效的学习方式

学习者反馈：会同时打开编辑器源码，当时大致能懂，但事后难以自己表达；原先讲解虽然细，却连续引入太多概念，缺少主动回忆与主题边界。

改为 small tasks + active recall：

1. 给一个具体业务问题，只看必要的几行源码。
2. 学习者先用自己的话回答，不要求标准术语。
3. 根据回答补足一个关键区别，再继续。
4. 每节留下简短结论；进入新主题前明确提示并确认是否继续。

用户明确反馈该方式比此前有效。继续以中文为主、保留约 30% 的技术英文，但不为了比例重复堆叠术语；不重新讲熟悉的基础 SQL。

### Recall record｜回答与校准

| Task | Learner response / Calibration |
| --- | --- |
| Ethereum 对应哪个模型？ | 回答 dex_ethereum_trades；校准它是 model name，对应表为 dex_ethereum.trades |
| 为什么读取它？ | 回答获取数据；补足目的：与其他链交易汇总 |
| Arbitrum 对应哪个模型？ | 正确回答 dex_arbitrum_trades |
| example 接入需要什么？ | 正确提出链名单加 example，并存在 dex_example_trades |
| 缺少 amount_usd 能否直接读取？ | 提出置空或后续补算方案；校准“改造方案”与“当前代码能否直接读取”的区别 |
| 字段存在但为 NULL 是否因缺字段报错？ | 正确回答不会 |
| 全 NULL 是否说明 enrichment 有问题？ | 正确拒绝“估值验证完成”；校准原因尚未确定，不能直接认定 Bug |
| 不同链同哈希与日志索引能否合并？ | 正确回答应分别保留，因为属于不同链 |
| 历史交易应属于哪个月份？ | 正确选择交易发生月份 2022-09-01 |

这些是本轮理解检查，不等于长期记忆或数据验证已经通过。

## 9. Next topic and validation boundary｜下次从哪里开始

先完成本次记录与审核。下次开始新主题前，明确说明将从 cross-chain integration 转入 partitioning and incremental updates。

先回顾 block_month，再用一个具体场景解释事件时间与处理时间、近期窗口与历史修正的区别；一次只引入一个问题。上一条历史修正问题尚未回答，不跳过，也不作为已掌握内容。

静态源码阅读与问答已完成；未安装软件、未运行 dbt deps / compile / test，未查询 Dune 数据、未执行数据库写操作、未修改上游 Spellbook。真实数据质量验证、Issue 复现及 PR 仍待开展。
