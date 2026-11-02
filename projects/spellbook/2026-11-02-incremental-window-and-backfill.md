# 2026-11-02｜增量窗口与历史回填：小课题收束

## 1. Scope and outcome｜本小课题范围

本节收束增量窗口、历史修正与 backfill 的概念讨论，不继续测试通用 SQL 或数据处理基础。

核心结论：**Read window first, merge second。读到历史修正，是它参与本次更新的前提；成功写回还取决于目标匹配条件。**

这是 conceptual review，不是 Spellbook 增量实现核查。尚未阅读 incremental_predicate() 的具体定义、生成 SQL、参数与窗口，也未确认项目真实的历史回填流程。

## 2. Example and conclusions｜练习场景与结论

下面是此前讲解的简化条件，不是仓库原文，也未执行：

```sql
WHERE block_time >= TIMESTAMP '2026-10-30 00:00:00'
```

假设一条交易的发生时间为 2022-09-28，amount_usd 从 NULL 修正为 100，但 block_time 没有变化。

| Question | Agreed conclusion |
| --- | --- |
| 修正后的记录能否通过上述时间条件？ | 不能；交易时间仍在源数据读取窗口之外 |
| 配置 merge 能否自动发现被筛掉的修正？ | 不能；本次 merge 没有收到该修正记录 |
| 只补处理 2022-09-28 是否比重跑全部历史更精准？ | 范围更明确；真实操作仍需核查项目支持方式 |
| 已读到历史修正，是否足以保证更新旧记录？ | 不足；若目标匹配范围排除了旧记录，仍无法匹配它 |

Source selection 与 target matching 是两个不同位置的条件。当前只说明了这个区别，没有检查 Spellbook 的实际目标谓词，也没有推断未匹配记录最终会被怎样处理。

Learning examples are not production evidence。上述数值、窗口和目标匹配场景均为练习，不表示发现了真实模型遗漏或 Bug。

## 3. Backfill clarification｜纠正任务前提

学习者指出：backfill 可以作为独立任务运行，日常任务仍按原流程处理。此前问题默认了“临时修改日常任务窗口”，但没有交代这个前提，造成歧义。

两种方式应分开描述：

- Separate backfill job：独立回填历史范围；日常任务的窗口不因而必然改变。
- Temporary window change：如果临时调整了原任务的读取范围，结束后应恢复正常配置。

独立任务不代表绝对没有影响。如果同时写同一目标表，仍需考虑写入冲突与资源竞争；本次未核查 Spellbook 的调度和并发机制。

Targeted backfill 是讨论中的策略名称，不是已确认的 Spellbook 命令或已实施操作。

## 4. Learning feedback｜已确认的讲解调整

学习者明确表示：SQL、数据关系、数据时间处理、merge 与 backfill 等通用技术已经熟练，不需要反复测试。之前“讲慢一点”不应被理解为对这些基础概念持续出选择题。

保留小步推进，但重新确定重点：

1. Project-specific knowledge：聚焦 Spellbook 的真实文件、宏调用、dbt 配置及生效机制。
2. No repetitive basic quizzes：不重复测试熟悉的通用数据处理原理，不为互动而提问。
3. Complete context：确需提问时交代运行方式、数据范围、源表或目标表条件、是否允许改代码，以及判断目标。
4. Explicit topic boundaries：进入新主题前明确提示，不直接用“下一步”切入新知识。
5. Chinese-first with technical English：保持必要技术英文，不为比例叠加同义标签。

这些更新细化并覆盖此前“每一步都主动判断”的过度应用，不否定短小、具体、基于源码的学习方式。

## 5. Completion gate｜小课题结束后的固定流程

用户确认以下节奏：

**一个小课题 → 总结已学内容与未解决问题 → 保存 Notion、展示 GitHub 本地 diff → 用户确认后 commit / push → 同步发布状态 → 再开始新课题。**

Notion 保存过程、理解校准和学习决定；个人 GitHub 仓库保存整理后的 Markdown、证据与进度。未同步完成前不推进新主题。

本次仅准备记录和本地审核稿。用户本次同意整理，不等于提前批准尚未展示的 diff；实际发布仍等待确认。

## 6. Next topic｜下一课，暂不展开

下一课核查 Spellbook 的 incremental_predicate()：

- 找到真实宏文件及 dex_trades 的调用位置。
- 核对宏具体生成什么条件、有哪些参数与运行分支。
- 区分用于源数据筛选的调用和用于目标表匹配的调用。
- 只依据实际源码说明项目行为，不用简化场景替代实现证据。

Next lesson starts only after this topic is summarized and synchronized。当前没有开始上述实现核查。

## 7. Verification boundary｜本次证据与未执行事项

本记录依据本轮对话、学习者反馈与前序已保存的跨链接入记录整理；本节没有新增执行层面的验证。

- Completed：概念讨论收束、任务前提校准、学习方式与同步流程确认。
- Pending：incremental_predicate() 源码核查、实际编译 SQL、真实 backfill 流程及数据结果。
- Not executed：dbt deps / compile / test、Dune 查询、数据库写入、历史回填。
- Upstream boundary：没有修改 C:\Dev\spellbook；个人记录保存在独立学习仓库。
