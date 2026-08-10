# dsh-tool-stat

DSH 统计工具插件 —— 描述统计、百分位数、频数分布、相关性计算。零依赖、纯函数、确定性。

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 动机

Agent 处理数值数据时，从 CSV / JSON 提取出数值数组后往往需要聚合分析（均值、分位数、分布、相关性）。当前工具链没有这类能力：`calculator` 只做单表达式求值，`dsh-tool-csv` 的 `stats` 只报告行/列结构信息——一组观测值的统计聚合是空白。

本插件接收显式传入的有限数值数组或成对观测值，提供确定性的统计计算，不读取文件、不访问网络、不创建进程、不保存状态，相同输入永远得到相同输出。

## 数值稳定性与错误契约

1. **补偿求和 / 稳定方差**：`sum` 用 Neumaier 补偿求和，`variance` 用 Welford 在线算法——大数值、近相等数值下不丢精度；
2. **有限数强约束**：拒绝 `NaN` / `Infinity`（带下标定位的错误信息）；`-0` 在输入与输出中均规范化为 `0`；
3. **溢出回检**：所有结果在返回前再次做有限数检查，任何中间/最终溢出返回 `numeric-overflow` 错误——canonical 输出**绝不含**非有限值；
4. **纯函数**：输入数组永不被修改（内部只读遍历 + 需要时先拷贝排序）；
5. **零方差语义**：`correlation` 遇零方差配对返回 `defined: false` + `reason: "zero-variance"`，而不是 NaN 或 ±Infinity；
6. **资源上限**：观测值 1..100,000、百分位请求 ≤ 100、distinct 输出 ≤ 10,000（超出按确定规则截断并标注）。

## 工具声明

注册 `stat` 工具（`@deepseek-ai/dsh-tool-stat`，row id `tool-stat`），统一输出 JSON 文本字符串。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `action` | string | ✅ | `describe` / `percentile` / `frequency` / `correlation` |
| `values` | array<number> | ✅ | 有限数值观测（1..100,000）；`-0` 归一化为 `0` |
| `other` | array<number> | | correlation 的配对观测；长度须与 `values` 相同（≥2） |
| `percentiles` | array<number> | | percentile 的百分位（0..100，1..100 项）；输出保持请求顺序，重复保留 |
| `method` | string | | 相关系数方法：`pearson`（默认）/ `spearman`（midrank 平均秩） |
| `sample` | boolean | | 方差分母：`true` 用样本（n-1），默认 `false`（总体 n） |

## Actions

| action | 功能 | 输出示例 |
|---|---|---|
| `describe` | count / sum / min / max / mean / median / variance / standardDeviation / q1 / q3 / iqr（Neumaier + Welford，population 或 sample） | `{"count":5,"sum":15,"mean":3,"median":3,"variance":2,...}` |
| `percentile` | 一个或多个百分位数，线性插值 `h=(n-1)*p`，输出按请求顺序、重复保留 | `{"percentiles":[{"percentile":50,"value":10},...]}` |
| `frequency` | 严格相等分组：value / count / ratio（分母为原始计数），升序输出 | `{"groups":[{"value":1,"count":2,"ratio":0.4},...]}` |
| `correlation` | Pearson 或 Spearman 相关系数；零方差返回 `defined:false` | `{"method":"pearson","defined":true,"value":1,"reason":null}` |

## 示例

```
stat { action: "describe", values: [1, 2, 3, 4, 5] }
  → {"action":"describe","count":5,"sum":15,"min":1,"max":5,"mean":3,"median":3,"variance":2,"standardDeviation":1.4142135623730951,"q1":2,"q3":4,"iqr":2,"sample":false}

stat { action: "percentile", values: [1, 5, 10, 20, 50], percentiles: [25, 50, 90] }
  → {"action":"percentile","count":5,"percentiles":[{"percentile":25,"value":5},{"percentile":50,"value":10},{"percentile":90,"value":38}]}

stat { action: "correlation", values: [1, 2, 3, 4], other: [2, 4, 6, 8], method: "pearson" }
  → {"action":"correlation","method":"pearson","count":4,"defined":true,"value":1,"reason":null}
```

> 详细设计（算法定义、输出协议）见 `DESIGN.md`（随代码上传）。
