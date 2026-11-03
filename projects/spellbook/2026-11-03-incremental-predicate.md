# 2026-11-03｜incremental_predicate 源码与两处调用

## 1. Outcome｜本小课题结论

学习时间：2026-11-02 至 2026-11-03。完成三项静态源码阅读：incremental_predicate() 的表达式、DEX 默认参数来源、dex_trades 中源与目标两处调用。

**同一个 SQL 条件生成宏，通过调用位置分别控制 source filtering 和 target matching。**

大纲位置：这是此前已约定的“Spellbook 增量实现”，衔接跨链接入和历史修正概念讨论；没有延伸为回填操作、调度设计或通用 SQL 训练。

## 2. Source map｜真实文件与对象

以下路径相对于 C:\Dev\spellbook。源码快照 HEAD：49d15619c3b6d679a69d7b6e2ca3d1022d68ca3d。本次未执行 dbt compile。

| File | Object type / Role |
| --- | --- |
| `dbt_macros/dune/incremental_predicate.sql` | Jinja macro：生成时间比较条件 |
| `dbt_subprojects/dex/dbt_project.yml` | Project configuration：定义 dbt variables 及环境变量默认值 |
| `dbt_subprojects/dex/models/trades/dex_trades.sql` | dbt model：源筛选调用、目标匹配配置与 CI 分支 |
| `dbt_macros/dune/incremental.sql` | Merge-generation macro：trino__get_merge_sql，将谓词和匹配键组装到 MERGE ON |

dbt model name 是 dex_trades；正常发布的 database relation 是 dex.trades。CI relation 可能在临时 schema 中。

## 3. Macro expression｜宏只负责生成条件

完整原文：

```text
{% macro incremental_predicate(column) -%}
{{column}} >= date_trunc('{{var("DBT_ENV_INCREMENTAL_TIME_UNIT")}}', now() - interval '{{var('DBT_ENV_INCREMENTAL_TIME')}}' {{var('DBT_ENV_INCREMENTAL_TIME_UNIT')}})
{%- endmacro -%}
```

输入包括调用者传入的 column，以及通过 var() 读取的两个 dbt variables。宏本身不读取表、不执行 merge，也不判断本次是不是增量运行。

当 column 为 block_time、参数为 3 / day 时，手工展开得到：

```sql
block_time >= date_trunc('day', now() - interval '3' day)
```

这是依据源码进行的 manual expansion，不是实际编译产物。date_trunc 将边界对齐到零点，并非精确的“当前时刻减 72 小时”。

## 4. Configuration source｜var() 与 env_var() 的连接

DEX 项目配置的 vars 下有：

```yaml
DBT_ENV_INCREMENTAL_TIME: "{{ env_var('DBT_ENV_INCREMENTAL_TIME', '3') }}"
DBT_ENV_INCREMENTAL_TIME_UNIT: "{{ env_var('DBT_ENV_INCREMENTAL_TIME_UNIT', 'day') }}"
```

链路是：环境变量（若设置）→ 项目 vars（否则使用默认 3 / day）→ 宏通过 var() 读取 → 生成条件。

宏调用的是 var()，项目配置才调用 env_var()。不能仅凭 DBT_ENV_ 这个名称就推断取值来源；本次通过配置原文确认了连接方式。

Repository defaults are not runtime evidence。本次确认的是仓库默认值，没有核查某次本地、CI 或生产运行的参数覆盖，也没有断言生产一定使用 3 / day。

## 5. Source filtering｜上游读取范围

dex_trades 的非 CI 分支中：

```text
{% if is_incremental() %}
WHERE {{ incremental_predicate('block_time') }}
{% endif %}
```

该条件位于各链模型的 FROM ref(...) 后面，筛选的是本次从上游各链读取的数据。

默认 3 / day 生效时，对应：

```sql
WHERE block_time >= date_trunc('day', now() - interval '3' day)
```

Scope matters：这里的前提是 target.name 不是 ci，并且 is_incremental() 为真。若该增量判断为假，这个分支不添加该 WHERE。

## 6. CI branch｜不能把所有运行方式都说成三天

模型先判断：

```text
{% if target.name == 'ci' -%}
WHERE block_date > current_date - interval '7' day
```

省略了原文中的解释注释，条件保持原样。CI 直接使用这段 block_date 条件；不走后面的源筛选 incremental_predicate() 调用。

因此：CI source window 与默认非 CI incremental source window 是不同分支。本次只核查源码，不把它解释为所有 CI 或生产执行行为都已验证。

## 7. Target matching｜目标表匹配范围

同一模型的 config 中有：

```text
incremental_predicates = [
    incremental_predicate('DBT_INTERNAL_DEST.block_time')
]
```

这里生成的条件不是上游 SELECT 的 WHERE。仓库中的 trino__get_merge_sql 宏先把 incremental_predicates 复制到 predicates，再追加 unique_key 等值匹配条件，最终合并到 MERGE ON。

源码中的 merge 结构：

```text
merge into {{ target }} as DBT_INTERNAL_DEST
    using {{ source }} as DBT_INTERNAL_SOURCE
    on {{"(" ~ predicates | join(") and (") ~ ")"}}
```

DBT_INTERNAL_DEST 是目标表别名；DBT_INTERNAL_SOURCE 是本次处理结果的别名。默认参数下，目标时间谓词手工展开为：

```sql
DBT_INTERNAL_DEST.block_time
    >= date_trunc('day', now() - interval '3' day)
```

This restricts matching, not deletion。它限制目标记录能否参与匹配，不会因为旧记录在范围外就自动删除它们。

本节只追踪了谓词进入 MERGE ON 的组装路径；没有运行 materialization、没有核验实际执行 SQL，也没有展开 merge_skip_unchanged 等其他机制。

## 8. Checkpoint｜已完成与未验证

| Item | Status |
| --- | --- |
| Macro expression | 已读源码 |
| DEX default variables | 已确认默认 3 / day 及 env_var() 来源 |
| Source call | 已定位并区分非 CI / incremental 条件 |
| Target call | 已追踪到仓库 MERGE ON 生成逻辑 |
| Runtime overrides | 未核查 |
| Actual compiled / executed SQL | 未生成、未执行 |
| Real Dune data quality / backfill | 未验证、未运行 |

本次未安装依赖、未运行 dbt deps / compile / test，未查询或写入数据库，未修改上游 Spellbook。

## 9. Close this topic｜完成后先同步，不立即扩展

沿用已确认的规则：小课题完成 → 总结 → Notion 保存与 GitHub 本地 diff → 用户确认后 commit / push → 更新发布状态 → 再进入新课题。

本次三个阅读目标已经完成。尚未开始下一个知识点，也不把实际回填或其他 merge 机制自动加入下一课。

后续回到既有大纲的 contribution readiness：在本篇审核发布后，再明确下一个小课题的范围。真实数据验证继续作为待办，不把源码阅读标成数据验证通过。
