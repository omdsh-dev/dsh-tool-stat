# dsh-tool-stat

DSH 统计工具插件 —— 描述统计、百分位数、频数分布、相关性计算。零依赖、纯函数、确定性。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 动机

Agent 处理数值数据时，从 CSV / JSON 提取出数值数组后往往需要聚合分析（均值、分位数、分布、相关性）。当前工具链没有这类能力：`calculator` 只做单表达式求值，`dsh-tool-csv` 的 `stats` 只报告行/列结构信息——一组观测值的统计聚合是空白。

本插件接收显式传入的有限数值数组或成对观测值，提供确定性的统计计算，不读取文件、不访问网络、不创建进程、不保存状态，相同输入永远得到相同输出。

## 用途

| action | 功能 |
|---|---|
| `describe` | 描述统计：count / sum / min / max / mean / median / variance / standardDeviation / q1 / q3 / iqr（Neumaier 补偿求和 + Welford 方差，数值稳定） |
| `percentile` | 计算一个或多个百分位数（线性插值，`0..100`） |
| `frequency` | 数值频数分布（严格相等分组、升序输出、ratio 分母为原始计数） |
| `correlation` | Pearson 或 Spearman 相关系数（含零方差处理） |

资源边界：最大 100,000 个观测值、100 个百分位数请求、10,000 个 distinct 输出项；拒绝 `NaN`/`Infinity`；所有结果在返回前再次做有限数检查（溢出返回明确错误，绝不让 canonical 输出含非有限值）。

> 详细设计（算法定义、输出协议、测试计划）见 `DESIGN.md`，将在完成开发后随代码上传。
