# 2026-10-30｜dbt seed tests 与学习路径复核

## 1. Learning outcome｜本节目标与结论

今天把单条 Swap 的人工核对，连接到自动化 seed test：理解模型与测试如何绑定、怎样匹配样本、如何判断通过，以及失败信息怎样帮助排查。

Testing chapter closed：测试专题的基础源码阅读告一段落。实际 dbt tests 和 Dune queries 均未执行；没有确认 Bug，没有创建 Issue / PR。

## 2. Source map｜文件与对象

以下路径相对于上游 Spellbook 仓库，不是个人日志仓库。

| File / Object | Role |
| --- | --- |
| `dbt_subprojects/dex/models/trades/ethereum/_schema.yml` | YAML configuration：为模型绑定测试与参数 |
| `dbt_subprojects/dex/tests/generic/check_dex_base_trades_seed.sql` | Generic test：指定定位字段与比较字段 |
| `dbt_macros/generic-tests/check_seed_macro.sql` | Jinja macro：生成匹配、计数、比较和失败输出 SQL |
| `dbt_subprojects/dex/seeds/trades/uniswap_ethereum_base_trades_seed.csv` | Seed CSV：维护者保存的预期样本 |
| `dbt_subprojects/dex/models/trades/ethereum/platforms/uniswap_v3_ethereum_base_trades.sql` | Model SQL：配置并调用交易宏 |
| `dbt_subprojects/dex/macros/models/_project/uniswap_compatible_trades.sql` | Trading macro：转换 Swap 并关联池信息 |

Model name：`uniswap_v3_ethereum_base_trades`。Database relation：`uniswap_v3_ethereum.base_trades`。文件名、dbt 模型名和数据库表名需要明确区分。

## 3. Test wiring｜测试如何连接模型

模型 YAML 中已有：

```yaml
- check_dex_base_trades_seed:
    arguments:
      seed_file: ref('uniswap_ethereum_base_trades_seed')
      filter:
        version: 3
```

这段配置位于模型的 `data_tests` 下。dbt 自动传入被测 `model`；`seed_file` 指向加载后的 seed relation；`filter` 只选择 seed 中的 version=3 样本，不是在过滤模型数据。

The test definition separates identity from expected values：

- Matching columns：`tx_hash`、`evt_index`，用来定位同一个事件。
- Check columns：`block_number`、`token_bought_address`、`token_sold_address`、`token_bought_amount_raw`、`token_sold_amount_raw`。
- Not checked here：symbol、decimals、换算后数量、amount_usd。

## 4. Execution logic｜匹配与通过标准

`matched_records` 先取得模型样本，再以 seed 为左侧执行 LEFT JOIN；这样即使模型缺失记录，预期样本也不会从检查中消失。

`matching_count_test` 按定位键分组，要求模型匹配数量为 1。Zero matches 表示未找到；multiple matches 表示不唯一，但 seed 定位键重复也会影响计数，不能直接认定模型重复。

`equality_tests` 逐字段比较：

```sql
model_value IS NOT DISTINCT FROM seed_value
```

这是 NULL-safe equality：两侧都为 NULL 也相等，因此不能替代 not_null test。

最终查询保留失败项：

```sql
WHERE equality_check IS DISTINCT FROM TRUE
```

Under default test configuration，返回 0 条失败记录才通过。每条选中样本都必须满足：exactly one match + all checked values equal。自定义 severity / thresholds 会影响 dbt 的最终状态，需要以执行配置为准。

## 5. Failure output｜怎样理解失败结果

| Output field | Meaning |
| --- | --- |
| `test_description` | 失败检查，例如 matched records count 或 equality test: token_bought_amount_raw |
| `result_model` | 模型实际值，或匹配数量 |
| `expected_seed` | seed 预期值，或预期数量 1 |
| `equality_check` | 是否满足该项条件 |
| `tx_hash`、`evt_index` | 用于定位失败事件 |

学习时用金额差 1 的例子说明字段不一致；这是 hypothetical example，不是实际测试结果。

一条样本完全缺失时，数量检查与多个字段检查都可能失败。Failure rows are not the number of affected trades。

## 6. Test scope｜唯一性、遗漏与样本覆盖

同一模型还配置了 `dbt_utils.unique_combination_of_columns`，检查 `tx_hash + evt_index`。其范围是实际被测 relation，可能是 CI 测试表，而不一定是生产全历史数据。

| Quality dimension | Question | Existing check / approach |
| --- | --- | --- |
| Existence | 预期事件有没有进入模型？ | seed matching count；源表与模型对照 |
| Uniqueness | 同一事件是否重复？ | unique combination test |
| Correctness | 字段值是否正确？ | seed equality tests |
| Coverage | 检查了哪些日期、资产与行为？ | 分析 seed 与实际检查范围 |

Seed test 只覆盖收录的样本；唯一性测试不能发现缺失记录。Both tests passing does not prove full completeness。

本地只读检查 version=3 的 CSV，确认：

- Sample count：5。
- Date coverage：全部为 2022-09-28。
- Block range：15633337–15633347。
- Sold token：全部为 WETH 地址 `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2`。
- Bought tokens：4 个不同地址。

These are local CSV findings, not model execution results。全部卖出 WETH 不代表只覆盖 amount0 的一个正负分支；还需结合池的 token0/token1 排序。

Regression testing 的目标是让已知样本在代码变更后仍保持预期。覆盖有限不自动构成 Bug，也不足以决定提交补充 seed 的 PR。

## 7. Diagnostic queries｜只读验证方案，未执行

模型用 Swap 池地址关联 PoolCreated：

```text
Swap.contract_address = PoolCreated.pool
```

实际宏采用 INNER JOIN，因此不匹配的源事件不会进入输出。这是 SQL 的保留条件，不是已发现数据遗漏。

### JOIN coverage：找不到池信息的事件

下面供 Dune editor 临时诊断；不是仓库现有测试，也不是已验证通过的 SQL。

```sql
SELECT
    t.evt_block_time,
    t.evt_tx_hash AS tx_hash,
    t.evt_index,
    t.contract_address AS pool_address
FROM uniswap_v3_ethereum.Pair_evt_Swap AS t
LEFT JOIN uniswap_v3_ethereum.Factory_evt_PoolCreated AS f
    ON f.pool = t.contract_address
WHERE t.evt_block_time >= TIMESTAMP '2022-09-28 00:00:00'
  AND t.evt_block_time < TIMESTAMP '2022-09-29 00:00:00'
  AND f.pool IS NULL
LIMIT 100
```

PoolCreated 保留全历史：今天的 Swap 可以发生在多年前创建的池中。LIMIT 100 只限制展示结果，不是总数，也不保证扫描量小。

### Model completeness：源事件在模型中是否存在

```sql
SELECT
    t.evt_block_time,
    t.evt_tx_hash AS tx_hash,
    t.evt_index,
    t.contract_address AS pool_address
FROM uniswap_v3_ethereum.Pair_evt_Swap AS t
WHERE t.evt_block_time >= TIMESTAMP '2022-09-28 00:00:00'
  AND t.evt_block_time < TIMESTAMP '2022-09-29 00:00:00'
  AND NOT EXISTS (
      SELECT 1
      FROM uniswap_v3_ethereum.base_trades AS m
      WHERE m.block_date = DATE '2022-09-28'
        AND m.tx_hash = t.evt_tx_hash
        AND m.evt_index = t.evt_index
  )
LIMIT 100
```

返回值只能叫 missing-record candidates。需要继续确认 expected scope、pool matching、data freshness，不能直接认定 Bug。返回 0 行也不证明字段正确或不存在重复。

Direct table names 仅用于临时诊断；编写正式 dbt 模型或测试时应使用 source() / ref()。以上两项都需要真实 Dune 数据查询权限，本次未运行。

## 8. Local reproduction｜可执行的本地只读检查

```powershell
Set-Location C:\Dev\spellbook
$v3Samples = Import-Csv dbt_subprojects/dex/seeds/trades/uniswap_ethereum_base_trades_seed.csv |
    Where-Object version -eq '3'
$v3Samples | Measure-Object
$v3Samples | Group-Object block_date
$v3Samples | Select-Object block_number, evt_index, token_bought_address, token_sold_address
Get-Content dbt_subprojects/dex/tests/generic/check_dex_base_trades_seed.sql
Get-Content dbt_macros/generic-tests/check_seed_macro.sql
```

No installation, no database execution, no upstream edits。未重新运行 dbt deps / compile / test；没有验证 Factory 源表覆盖、模型输出、美元金额或全历史质量。

## 9. Learning-path decision｜已确认的路径调整

用户确认：主线一致，但测试后半段扩展到了覆盖率、诊断 SQL 和潜在测试贡献；这些属于 optional extensions，不继续无限展开。

1. Close the testing chapter：测试专题基础源码阅读结束，不代表测试通过。
2. Return to cross-chain lineage：下一课为“Ethereum 的交易记录，怎样进入跨链 dex.trades？”
3. One model at a time：明确真实文件路径、dbt model、database relation，不重复讲熟悉的基础 SQL。
4. Contribution readiness：架构主线读完后，学习筛选首次贡献任务、阅读 Issue 与 CI。
5. Validation stays pending：真实 Dune 数据检查继续待办，不因权限限制阻断本地学习。
6. Communication preference：中文为主，约 30% 技术英文；按具体文件与业务问题小步推进。

这次记录不占用预留的 03-issue-reproduction.md：目前没有复现 Issue，也未决定补充 seed。
