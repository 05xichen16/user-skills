# spark-log-analysis 使用说明

自动分析 YARN Spark 应用日志目录，输出结构化分析报告并生成 Markdown 文档。

---

## 快速开始

直接告诉 Claude 日志目录路径即可触发：

```
分析一下 logs (55) 这个目录的 Spark 日志
```

```
帮我看看 /data/spark-logs/application_xxx 的日志，有没有什么问题
```

```
这个任务跑了很久，帮我诊断一下 logs (56) 的执行情况
```

---

## 触发关键词

以下表达均会触发本 skill：

| 场景 | 示例 |
|------|------|
| 日志分析 | "分析日志"、"看看这个日志"、"日志诊断" |
| 性能问题 | "为什么跑这么慢"、"性能分析"、"找一下瓶颈" |
| Executor 问题 | "executor 使用率"、"task 分布"、"节点参与情况" |
| 报告生成 | "生成分析报告"、"输出分析文档" |
| 调参准备 | "调参建议"、"怎么优化这个任务" |

---

## 输入要求

### 必须提供
- **日志目录路径**：包含 `driver/` 和 `executor/` 子目录的 YARN 聚合日志目录

### 目录结构（标准格式）
```
logs (55)/
├── driver/
│   └── <node-name>/
│       └── <container-id>          ← Driver / AM 日志
└── executor/
    ├── <node-name>/
    │   └── <container-id>          ← Executor 日志（每节点一个）
    ├── <node-name>/
    │   └── <container-id>
    └── ...
```

> 其中位于 `executor/` 下的 `container_xxx_000001` 通常同时承担 AM 角色，
> 是信息最完整的日志文件。

---

## 输出内容

### 对话中输出结构化报告，包含：

| 章节 | 内容 |
|------|------|
| 基本信息 | Application ID、任务名称、用户、运行模式、时间范围 |
| 执行概览 | Job / Stage / Task 总数，耗时 Top 3 Stage |
| 资源配置 | 内存、核数、Executor 数量、并行度、GC 策略 |
| 数据处理量 | 输入文件数、分区数、Shuffle、Spill |
| **Executor 节点参与情况** | 节点映射表、Task 分配统计、冷启动耗时、负载均衡评估 |
| 数据倾斜分析 | 关键 Stage 内 Task 耗时分布，最长/最短比 |
| 异常与警告 | 按 CRITICAL / HIGH / MEDIUM / LOW 分级 |
| GC 状况 | GC 总耗时、占比、Full GC 次数、评估结论 |

### 同时在日志目录下生成 Markdown 文件：
```
<日志目录>/spark-analysis-report.md
```

---

## 能识别的问题类型

| 问题类型 | 识别方式 |
|---------|---------|
| 真实数据倾斜 | 同 Stage 内 Task 耗时最长/最短比 > 3x |
| 空分区浪费 | 大量 Task 耗时 < 100 ms，但分区数远超实际数据量 |
| OOM / Executor 丢失 | OutOfMemoryError、GC overhead、Executor lost |
| GC 压力 | GC 总耗时占任务时间 > 10% |
| 冷启动开销过大 | Executor 启动时间占总运行时间比例过高 |
| NameNode HA 延迟 | Operation category READ is not supported in state standby |
| Python 依赖路径问题 | RuntimeWarning: Failed to add file to Python path |
| 废弃配置 | has been deprecated 告警汇总 |
| 网络 / Shuffle 异常 | fetchFailed、Connection refused、Timeout |

---

## 使用示例

### 示例 1：基础分析

**输入：**
```
分析 logs (55) 目录下的 Spark 日志
```

**输出：** 完整结构化报告 + 生成 `logs (55)/spark-analysis-report.md`

---

### 示例 2：结合调参

**输入：**
```
分析 logs (56) 的日志，我可以调整的参数有
executor-cores、executor-memory、num-executors 和 num_partitions
```

**输出：** 完整报告 + 针对这 4 个参数的具体建议（含当前值 vs 建议值对照表）

---

### 示例 3：问题诊断

**输入：**
```
这个任务跑了 2 分半，感觉很慢，帮我看看 logs (56) 有没有异常
```

**输出：** 重点突出瓶颈 Stage 和数据倾斜情况，给出针对性优化方向

---

## 注意事项

- 日志文件超过 256 KB 时，skill 会自动改用 grep 精准提取，不会整体读取
- 若某项信息在日志中不存在，报告中标注"未找到"，不会猜测
- AM 日志（container_000001）是主要分析来源；Executor 日志用于补充各节点 Task 详情
- 同一应用的多个日志目录（如 logs(55) 和 logs(56)）分析后可自动做横向对比

---

## 文件位置

```
.claude/skills/spark-log-analysis/
├── SKILL.md      ← skill 核心指令（Claude 执行依据）
└── README.md     ← 本使用说明
```
