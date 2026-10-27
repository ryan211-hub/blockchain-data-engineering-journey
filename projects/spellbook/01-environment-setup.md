# 01 · Windows 开发环境搭建

## 完成范围

已完成 Windows 上的 Spellbook 本地开发环境搭建、依赖安装和 dbt 编译。本文按本次提供的已完成结果整理；本次文档初始化未重新执行环境安装、版本调整、`dbt deps` 或 `dbt compile`。

- Spellbook 开发目录：`C:\Dev\spellbook`
- 个人学习文档目录：`C:\Dev\blockchain-data-engineering-journey`
- 当前阶段：本地开发工具与编译流程已打通。

## 环境版本

| 工具 | 已完成配置的版本 | 用途 |
| --- | --- | --- |
| Python | 3.13.5 | Python 运行环境 |
| Git | 2.53.0 | 版本管理与仓库协作 |
| uv | 0.13.6 | Python 环境与依赖管理 |
| dbt-core | 1.12.4 | dbt 项目解析与编译 |
| dbt-trino | 1.10.3 | Trino adapter |
| Codex 扩展 | 26.1007.21434 | 已降级至此版本，用于当前开发环境 |

以上是此次环境搭建记录中的版本，不作为后续升级建议。Codex 扩展降级的具体错误与原因未在本次材料中提供，因此不推断故障原因。

## dbt profiles-dir 问题

环境搭建过程中遇到了 dbt `profiles-dir` 配置目录定位问题，处理后 `dbt deps` 和 `dbt compile` 均已成功。

复现时应确认 dbt 使用的 profiles 目录包含实际的 `profiles.yml`。`--profiles-dir` 指向目录，`--project-dir` 指向 dbt 项目目录，两者用途不同。需要显式指定时，可使用 `--profiles-dir "<实际 profiles.yml 所在目录>"`；此处为参数示意，不是已执行命令。

本次材料未提供原始报错、最终 profiles 绝对路径和完整执行命令，因此这里不补写未经确认的细节。个人文档仓库不收录配置文件中的凭据。

## 已完成的验证

| 检查项 | 记录结果 | 说明 |
| --- | --- | --- |
| dbt deps | 成功，退出码 0 | 依赖安装阶段已通过 |
| dbt compile | 成功，退出码 0 | 本地模型编译阶段已通过 |
| manifest.json | 存在 | 已生成 dbt 项目清单产物；本次材料未提供具体路径 |

退出码 0 和 `manifest.json` 存在是本次提供的历史执行结果，本次文档初始化未重新验证这些结果。

## 验证边界

`dbt compile` 成功说明本地编译流程已通过。它不等于模型已在 Dune 上执行，不等于已查询真实链上数据，也不等于行数、唯一键、业务指标或数据质量已验证通过。

后续需要单独记录实际执行的 SQL、查询范围、测试结果和数据核对证据，才能确认模型运行与数据质量情况。
