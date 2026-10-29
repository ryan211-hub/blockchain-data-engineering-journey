# Uniswap V3 seed 与链上 Swap 人工核对

学习日期：2026-10-29。

- [Notion 完整学习过程](https://app.notion.com/p/40819faa63d68158b992c1484d40ff30)
- [前序：DEX 模型分析](02-dex-model-analysis.md)
- 本地 Spellbook HEAD：`49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d`。

## 1. 目标与证据来源

选取仓库的一条 Uniswap V3 seed，将预期输出与真实 Swap 的事件字段、池资产及 token metadata 人工对应起来。

助手读取本地 CSV 和 YAML；学习者在 Etherscan 查看日志、读取合约，并把结果贴回对话。助手未能直接读取交易页面。下文链上字段属于学习者提供的浏览器读取结果，不写成助手独立抓取的链上数据。

学习者当前无法在 Dune 执行查询，本次改用区块浏览器。不将这一账户现状写成所有 Free 用户的统一政策。

## 2. 样本及测试的位置

以下路径相对于 `C:\Dev\spellbook`：

| 文件 | 职责 |
| --- | --- |
| `dbt_subprojects/dex/seeds/trades/uniswap_ethereum_base_trades_seed.csv` | seed 预期结果，选择 version=3 第一条 |
| `dbt_subprojects/dex/models/trades/ethereum/_schema.yml` | 给 `uniswap_v3_ethereum_base_trades` 模型声明 seed test |
| `dbt_subprojects/dex/tests/generic/check_dex_base_trades_seed.sql` | 通用测试入口，指定匹配与比较字段 |
| `dbt_macros/generic-tests/check_seed_macro.sql` | 底层比较逻辑，留作下一次细读 |

测试声明不在模型 SQL 中，而是按 YAML 的模型名称绑定：

```yaml
- check_dex_base_trades_seed:
    arguments:
      seed_file: ref('uniswap_ethereum_base_trades_seed')
      filter:
        version: 3
```

seed 是维护者预先保存的期望值，不是本次运行模型得到的数据。

## 3. 选定的交易与预期结果

[Etherscan 交易与日志](https://etherscan.io/tx/0x021a4ea590baee1e6bec5ecdb3b03cb67a4efaac67ac764f97ac3b152fe1ad10#eventlog)。

| 字段 | seed 预期值 |
| --- | --- |
| blockchain / project / version | ethereum / uniswap / 3 |
| block_date | 2022-09-28 |
| block_number | 15633347 |
| tx_hash | `0x021a4ea590baee1e6bec5ecdb3b03cb67a4efaac67ac764f97ac3b152fe1ad10` |
| evt_index | 8 |
| token_bought_address | `0xae78736cd615f374d3085123a210448e74fc6393` |
| token_sold_address | `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2` |
| token_bought_amount_raw | 490796462724159171 |
| token_sold_amount_raw | 510288592522475474 |

## 4. Swap Event 与池资产核对

学习者找到日志并提供：

```text
Name: Swap
Address: 0xa4e0faa58465a2d369aa21b3e42d43374c6f9613
amount0 (int256): -490796462724159171
amount1 (int256):  510288592522475474
```

`Address` 是发出日志的池合约，对应模型的 `project_contract_address`。学习者另确认区块号为 `15633347`、日志索引为 `8`，均与 seed 相符。事件索引不是该交易的第八次 Swap，也不是按页面顺序数第八条。

打开[池合约](https://etherscan.io/address/0xa4e0faa58465a2d369aa21b3e42d43374c6f9613#readContract)，进入 Contract → Read Contract；学习者读取：

| 字段 | 合约返回值 | 与 seed 的关系 |
| --- | --- | --- |
| token0 | `0xae78736Cd615f374D3085123A210448E74Fc6393` | 与买入资产地址一致 |
| token1 | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | 与卖出资产地址一致 |

token0/token1 按地址排序，编号不随买卖方向变化；amount0 固定属于 token0，amount1 固定属于 token1。大小写展示差异不改变地址字节值。

本次 `amount0 < 0`，因此按模型逻辑买入 token0、卖出 token1。两侧取绝对值后：

| 模型字段 | 从事件推导的值 | seed 核对 |
| --- | --- | --- |
| token_bought_amount_raw | 490796462724159171 | 一致 |
| token_sold_amount_raw | 510288592522475474 | 一致 |

本次通过池 getter 确认资产，没有直接核对 Factory PoolCreated 日志或 Dune 解码源表 JOIN。

## 5. Token metadata 与数量换算

分别打开 [token0 合约](https://etherscan.io/address/0xae78736Cd615f374D3085123A210448E74Fc6393#readContract)及 [token1 合约](https://etherscan.io/address/0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2#readContract)，读取 symbol、decimals。学习者提供：

| 方向 | symbol | decimals | 返回类型 | 换算后数量 |
| --- | --- | --- | --- | --- |
| 买入 token0 | rETH | 18 | uint8 | 0.490796462724159171 |
| 卖出 token1 | WETH | 18 | uint8 | 0.510288592522475474 |

```text
实际 token 数量 = raw amount / 10^decimals

买入 rETH = 490796462724159171 / 10^18
          = 0.490796462724159171

卖出 WETH = 510288592522475474 / 10^18
          = 0.510288592522475474
```

不是直接除以 decimals（18），而是除以 `10^18`，相当于小数点左移 18 位。uint8 是返回类型，换算参数为 18。

上述为精确十进制数学换算，不代表已经核实 DuneSQL 实际表达式的浮点精度或执行结果。今天从 token 合约读取 metadata，Spellbook enrichment 从 `tokens.erc20` 读取，不能直接等同。

## 6. 结果与限制

基于学习者提供的事件、池信息及确认，样本的交易定位、区块号、买卖地址和 raw amount 一致。可读为：卖出 **0.510288592522475474 WETH**，买入 **0.490796462724159171 rETH**。

| 检查 | 本次状态 |
| --- | --- |
| 本地 seed、YAML 声明读取 | 已完成 |
| 单条事件与 seed 人工核对 | 已完成；浏览器读取结果由学习者提供 |
| symbol / decimals 与数量换算 | 已完成；合约返回值由学习者提供 |
| Dune 解码表、tokens.erc20、模型实际输出 | 未查询 |
| PoolCreated 源表 JOIN 覆盖 | 未验证 |
| 价格、trusted 名单、amount_usd | 未验证 |
| dbt test、全量质量与增量运行 | 未执行 |
| Bug、Issue、PR | 未确认、未创建 |

单条 seed 的链上人工核对不等于 dbt test 通过，也不证明全量模型正确。本次未安装依赖、重新 compile 或修改上游模型。

## 7. 可复现的本地只读命令

```powershell
Set-Location C:\Dev\spellbook
Import-Csv dbt_subprojects/dex/seeds/trades/uniswap_ethereum_base_trades_seed.csv |
    Where-Object version -eq '3' |
    Select-Object -First 1 |
    Format-List

Select-String -Path dbt_subprojects/dex/models/trades/ethereum/_schema.yml -Pattern 'name: uniswap_v3_ethereum_base_trades' -Context 0,20
Get-Content dbt_subprojects/dex/tests/generic/check_dex_base_trades_seed.sql

[decimal]'490796462724159171' / [decimal]'1000000000000000000'
[decimal]'510288592522475474' / [decimal]'1000000000000000000'
```

命令分别复现本地样本读取、测试声明查看和精确数量计算；不自动复现 Etherscan 网页观察。

## 8. 下一步

逐段阅读 seed test 的比较实现：如何定位模型记录、检查匹配次数、比较字段、输出失败结果。先在本地阅读即可；实际执行 tests 仍需要数据访问，且需考虑 `store_failures` 的写入行为。

本文是单条 Swap 的人工核对记录，不是 Issue 复现，因此不占用规划中的 `03-issue-reproduction.md`。
