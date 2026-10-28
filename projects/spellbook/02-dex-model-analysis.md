# DEX Swap 模型阅读：Uniswap V3 到美元估值

记录日期：2026-10-28。范围：本地源码与已有编译 SQL 的只读分析。

- [Notion 完整学习记录](https://app.notion.com/p/40719faa63d681a4bc4ce7e6be731017)
- 上游仓库：[Dune Spellbook](https://github.com/duneanalytics/spellbook)
- 本次读取的上游本地 HEAD：`49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d`。以下源码链接固定到该提交。
- 本次没有重新运行 dbt compile，没有执行 Dune 查询、dbt tests 或数据库写操作，也没有修改 Spellbook 上游仓库。
- 本篇完成的是核心代码阅读，不代表已验证真实链上数据，也没有确认 Bug 或提交 Issue / PR。

## 1. 文件、dbt 模型与数据库表

SQL 文件去掉 `.sql` 后是本次涉及的 dbt 模型名；数据库关系名由 `schema`、`alias` 等配置决定。下表为正常发布名称，CI 等环境可能使用不同 catalog 或 schema。

| SQL 文件路径（相对 Spellbook 根目录） | dbt 模型名 | schema.alias |
| --- | --- | --- |
| `dbt_subprojects/dex/models/trades/ethereum/platforms/uniswap_v3_ethereum_base_trades.sql` | `uniswap_v3_ethereum_base_trades` | `uniswap_v3_ethereum.base_trades` |
| `dbt_subprojects/dex/models/trades/ethereum/dex_ethereum_base_trades.sql` | `dex_ethereum_base_trades` | `dex_ethereum.base_trades` |
| `dbt_subprojects/dex/models/trades/ethereum/dex_ethereum_trades.sql` | `dex_ethereum_trades` | `dex_ethereum.trades` |
| `dbt_subprojects/dex/models/trades/dex_trades.sql` | `dex_trades` | `dex.trades` |

```text
Swap Event + PoolCreated Event
  → uniswap_v3_ethereum_base_trades：协议事件转标准基础记录
  → dex_ethereum_base_trades：汇总平台，补交易字段，去重
  → dex_ethereum_trades：补 token metadata、数量、美元价值
  → dex_trades：汇总各链 trades（本次仅定位入口）
```

`_projects/uniswap/ethereum/uniswap_v3_ethereum_trades.sql` 是读取 `dex_ethereum_trades` 的下游 view，不是原始 Swap 解析入口。当前 `dex_trades` 自身为 incremental 模型，不应按通用架构示意误写成 view。

## 2. 模型调用宏，而不是把完整 SQL 都写在入口

[Uniswap V3 模型入口](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/models/trades/ethereum/platforms/uniswap_v3_ethereum_base_trades.sql)调用共享宏：

```jinja
uniswap_compatible_v3_trades(
    blockchain = 'ethereum'
    , project = 'uniswap'
    , version = '3'
    , Pair_evt_Swap = source('uniswap_v3_ethereum', 'Pair_evt_Swap')
    , Factory_evt_PoolCreated = source('uniswap_v3_ethereum', 'Factory_evt_PoolCreated')
)
```

实现位于 [uniswap_compatible_trades.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/macros/models/_project/uniswap_compatible_trades.sql)，宏名是 `uniswap_compatible_v3_trades`，不是文件名。

- `source()` 引用已声明的 Dune 数据源，本例为已解码事件；不负责解码原始日志。
- `ref()` 引用本仓库 dbt 模型的输出，并登记模型依赖；本身不会立即执行上游模型。本例的 Uniswap V3 base 入口没有使用 `ref()`，下游汇总模型使用它。
- DEX 的 `macro-paths` 包括 `macros` 和 `../../dbt_macros`；dbt 按宏名加载并在编译时展开模板，不需要手写 import。
- 宏不是独立数据库表；CTE 也不是独立 dbt 模型。

## 3. Swap 事件如何转换为基础交易

Swap 提供 `amount0/amount1`；PoolCreated 提供对应资产地址。宏中的关联为：

```jinja
INNER JOIN {{ Factory_evt_PoolCreated }} f
    ON f.{{ pair_column_name }} = t.contract_address
```

参数 `pair_column_name` 默认是 `pool`，展开后为 `ON f.pool = t.contract_address`。

`amount0` 始终对应 `token0`，`amount1` 始终对应 `token1`。编号不会随买卖方向改变；正负号是池子的余额变化视角。

| 条件 | 买入资产 | 买入 raw amount | 卖出资产 | 卖出 raw amount |
| --- | --- | --- | --- | --- |
| `amount0 < 0` | `token0` | `abs(amount0)` | `token1` | `abs(amount1)` |
| 否则 | `token1` | `abs(amount1)` | `token0` | `abs(amount0)` |

输出增加链、协议、版本、日期字段，将金额转为 `UINT256`，保留 `tx_hash` 和 `evt_index`。`taker` 使用事件 `recipient`，不保证是最终用户钱包；本次调用的 `maker` 为 NULL。宏内部取了 `f.fee`，但最终 base 输出不包含 fee。

`tx_hash + evt_index` 是本模型唯一键。一笔交易可产生多个 Swap，因此不能仅按 `tx_hash` 去重。该层尚未补充 symbol、decimals、实际数量或美元价值。

## 4. Ethereum 汇总、补字段与去重

[dex_ethereum_base_trades.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/models/trades/ethereum/dex_ethereum_base_trades.sql)通过 `base_models` 列表引用各平台 base 模型，并调用 [dex_base_trades_macro.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/macros/models/dex_base_trades_macro.sql)。

Jinja 循环在编译阶段生成多段 SELECT 与 `UNION ALL`。各协议已经统一字段，因此该层不重新解释 `amount0/amount1`。列表中的来源既有 AMM，也有其他交易执行机制；并非每项都代表独立交易所公司。

随后通过 [add_tx_columns.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_macros/shared/add_tx_columns.sql)关联 `ethereum.transactions`，匹配日期、区块号及交易哈希：

| 新增字段 | 含义 |
| --- | --- |
| `tx_from` | 整笔 Ethereum 交易发送者 |
| `tx_to` | 整笔交易直接调用的目标，不一定是 Swap 池 |
| `tx_index` | 交易在区块中的位置，与事件索引 `evt_index` 不同 |

最终按 `project, tx_hash, evt_index` 分组，保留 `duplicates_rank = 1`。它不会合并事件索引不同的多个 Swap。`version` 未参与该去重；组内 `order by tx_hash` 也没有定义来源优先级。

潜在重复可能来自模型覆盖重叠或上游重复，但本次没有验证真实重复。当前普通 Uniswap V4 路径已配置排除 Angstrom hook，并走专用解析路径，不能以此断言当前发生重复。去重规则与数据冲突可作为后续调查方向，本次未认定架构缺陷。

## 5. Token metadata 与实际数量

[dex_ethereum_trades.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/models/trades/ethereum/dex_ethereum_trades.sql)调用 [enrich_dex_trades.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/macros/models/enrich_dex_trades.sql)。Balancer V3 另走专用路径；本次关注普通路径。

宏按链筛选 `tokens.erc20`，分别用买入、卖出地址做两次 LEFT JOIN，取得各自 symbol 与 decimals：

```sql
token_bought_amount_raw / power(10, erc20_bought.decimals)
token_sold_amount_raw / power(10, erc20_sold.decimals)
```

示意：USDC 的 raw amount 为 `100000000`，若 decimals 为 6，则为 100 USDC。买卖两侧使用各自 decimals；raw amount 与换算后 amount 同时保留。元数据缺失不会直接删除交易，但 symbol 和换算后 amount 可能为 NULL。

## 6. 美元交易金额与 trusted 名单

[add_amount_usd_dex_trades.sql](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/macros/add_amount_usd_dex_trades.sql)读取已有单价，计算一条 Swap 的 `amount_usd`，不生成 token 美元单价。

- `source('prices', 'usd_with_native')`：按链、token 地址、交易所在分钟关联价格；不是查找最近的可用价格。
- `source('prices', 'trusted_tokens')`：该宏只读取 `blockchain`、`contract_address`，用于估值优先级。source YAML 将其映射到 `prices_v2` schema，逻辑 source 名不一定等于实际数据库 schema。
- 名单完整字段、真实成员、入选标准本次均未验证，不能宣称 USDC / WETH 已核实在名单中，也不把它理解成投资安全认证。

`COALESCE` 顺序为：

1. trusted 买入 token 的数量 × 买入单价。
2. trusted 卖出 token 的数量 × 卖出单价。
3. 买入数量 × 买入单价。
4. 卖出数量 × 卖出单价。

示意：支付 100 USDC、收到 0.05 WETH，若单价分别为 1 美元、2000 美元，两侧均可估值为 100 美元。两侧衡量同一次兑换，选择一个结果，不相加成 200 美元。两侧估值可能不同，也不用于判断用户盈亏。均不可用时，交易仍保留，`amount_usd` 为 NULL。

美元金额统一计量单位后，才能汇总不同 token 交易的成交量。

## 7. 已定位的测试与待验证事项

Uniswap V3 base 模型的 [schema YAML](https://github.com/duneanalytics/spellbook/blob/49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d/dbt_subprojects/dex/models/trades/ethereum/_schema.yml) 声明：

| 测试 | 覆盖 |
| --- | --- |
| `dbt_utils.unique_combination_of_columns` | `tx_hash, evt_index` 唯一性 |
| `check_dex_base_trades_seed` | 共享 seed 中 `version=3` 的 5 条样本 |

seed 按交易哈希和事件索引匹配，检查样本匹配次数、区块号、两侧 token 地址和 raw amount。该模型条目未声明键列 `not_null` tests。配置已阅读，测试未执行，样本不等于全量覆盖。

需真实 Dune 数据才能检查：池创建 INNER JOIN 覆盖率与唯一性、交易表 INNER JOIN 覆盖率、金额与事件一致性、去重前的数据冲突，以及超出默认约 3 天增量回看窗口的迟到事件。PoolCreated 保留全历史，不能仅保留近期新建池。

DEX 配置启用 `store_failures`，不将直接运行 `dbt test` 当作严格只读替代。

## 8. 可复现的本地只读查看

以下命令读取源码、CSV 或已有编译产物，不安装依赖、不重新编译、不查询数据库：

```powershell
Set-Location C:\Dev\spellbook
Get-Content dbt_subprojects/dex/models/trades/ethereum/platforms/uniswap_v3_ethereum_base_trades.sql
Get-Content dbt_subprojects/dex/macros/models/_project/uniswap_compatible_trades.sql | Select-Object -Skip 55 -First 76
Get-Content dbt_subprojects/dex/macros/models/dex_base_trades_macro.sql
Get-Content dbt_macros/shared/add_tx_columns.sql
Get-Content dbt_subprojects/dex/macros/models/enrich_dex_trades.sql
Get-Content dbt_subprojects/dex/macros/add_amount_usd_dex_trades.sql
Get-Content dbt_subprojects/dex/target/compiled/dex/models/trades/ethereum/platforms/uniswap_v3_ethereum_base_trades.sql
Import-Csv dbt_subprojects/dex/seeds/trades/uniswap_ethereum_base_trades_seed.csv | Where-Object version -eq '3'
```

本次已读取 Uniswap V3 的已有编译 SQL，其中可见展开后的池 JOIN、买卖方向与 UINT256 转换。它只支持本地模板展开检查，不能证明 SQL 已在真实 Dune 数据中执行或 tests 已通过。命令中的编译文件要求本地已有 DEX 编译产物。

## 9. 下一步

- 选择一条真实 Swap 或短日期窗口，在 Dune 上核对事件、池信息、base 输出及最终金额。
- 记录查询 SQL、执行时间、结果与异常；根据证据决定是否进一步调查 Issue。
- `dex.trades` 跨链汇总实现与 tests 详细运行可留作后续学习。
- 本次没有 Issue / PR；遵循先展示本地 diff、审核后再 commit/push 的发布流程。
