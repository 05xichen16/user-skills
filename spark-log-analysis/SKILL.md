---
name: spark-log-analysis
description: >
  分析 YARN Spark 应用日志目录，输出结构化分析报告并生成 Markdown 文档。
  当用户提供 Spark / YARN 日志目录路径，并要求"分析日志"、"生成分析报告"、
  "查看性能"、"日志诊断"、"调参建议"时，立即触发本 skill。
  关键词包括：spark log、executor log、driver log、yarn log、application log、
  日志分析、性能分析、executor 使用率、task 分布。
  即使用户只说"分析一下这个日志目录"也应主动触发。
---

# Spark 日志分析 Skill

从 YARN Spark 应用日志目录中提取关键信息，识别异常，统计 Executor 使用情况，
并输出结构化报告（同时在日志根目录生成 Markdown 文件）。

---

## 第一步：探索目录结构

```bash
# 1. 列出日志根目录内容
ls "<LOG_DIR>"

# 2. 递归列出所有文件
find "<LOG_DIR>" -type f | sort

# 3. 统计各文件行数与字节数
wc -l <all_files>
wc -c <all_files>
```

识别以下角色：
- **Driver / AM 日志**：位于 `driver/` 子目录下（或 `executor/` 下 container_000001）
- **Executor 日志**：位于 `executor/` 子目录下，每个节点一个目录

> 文件超过 256 KB（约 2000 行）时，禁止整体 Read，改用 grep 精准提取。

---

## 第二步：提取基本信息

在 AM/Driver 最大日志文件中执行以下 grep（用 Bash 工具）：

```bash
BASE="<AM_LOG_FILE>"

# Application ID、用户、App 名称
grep -n "application_\|SPARK_APP_NAME\|ApplicationName\|appName\|dataops\|Running as user" "$BASE" | head -5

# 时间范围（首条和末条时间戳）
head -10 "$BASE"
tail -5 "$BASE"

# 运行模式（YARN / local / k8s）
grep -n "YarnClusterScheduler\|yarn\|local\[" "$BASE" | head -3
```

提取：Application ID、App 名称、提交用户、开始时间、结束时间、总耗时。

---

## 第三步：提取资源配置

```bash
# Executor 内存、核数、数量
grep -n "Will request\|executor resources\|MemoryStore started\|memoryOverhead\|cores.*amount\|memory.*amount" "$BASE" | head -10

# 动态资源分配
grep -n "dynamicAllocation\|dynamic.*allocation" "$BASE" | head -5

# 执行器启动命令（含 -Xmx、--cores 参数）
sed -n '200,340p' "$BASE"
```

提取：executor-memory、executor-cores、num-executors、MemoryStore 容量、GC 策略、动态分配开关。

---

## 第四步：提取 Job / Stage / Task 执行概览

```bash
# 所有 Stage 完成记录（含耗时）
grep -n "finished in" "$BASE"

# 所有 Job 完成记录
grep -n "Job.*finished:" "$BASE"

# 各 Stage 提交的 Task 数
grep -n "Submitting.*missing tasks" "$BASE"

# GC 耗时（按 Job）
grep -n "totalGCTime" "$BASE"
```

整理：
- Job 总数 / 成功数 / 失败数
- Stage 总数（实际执行 + 缓存复用）
- Task 总数（汇总 `Submitting X missing tasks` 中的 X）
- 耗时 Top 3 Stage
- 各 Job GC 耗时，累计 GC 总时间，GC 占比

---

## 第五步：统计 Executor 节点参与情况

```bash
# Executor 注册记录（ID → 节点映射）
grep -n "Registered executor\|Starting executor ID\|Launching container" "$BASE" | head -20

# 各 Executor 任务完成数（从 AM 日志中统计）
grep "Finished task" "$BASE" | sed 's/.*on \(node-group-[^ ]*\) (executor \([0-9]*\)).*/\1 exec\2/' | sort | uniq -c | sort -rn

# 各 Executor 节点日志：首个 Task 时间、末个 Task 时间、Task 总数
for EXEC_LOG in <EXECUTOR_LOG_FILES>; do
  echo "=== $(basename $(dirname $EXEC_LOG)) ==="
  grep "MemoryStore started\|Starting executor ID\|registered with driver" "$EXEC_LOG" | head -2
  grep "Finished task" "$EXEC_LOG" | head -1   # 首个
  grep "Finished task" "$EXEC_LOG" | tail -1   # 末个
  grep -c "Finished task" "$EXEC_LOG"           # 总数
done
```

输出表格：Executor ID → 宿主节点 → Container ID → 角色 → Task 数 → 占比 → 首末时间 → 活跃时长。

同时计算：
- 冷启动耗时（AM 启动时间 → 首个 Task 下发时间）
- 负载不均衡程度（最多 / 最少 Task 之比）

---

## 第六步：数据倾斜分析

```bash
# 分析关键 Stage 内各 Task 耗时分布（选耗时最长的前 3 个 Stage）
grep "stage X.0" "$BASE" | grep "Finished task" | head -50
```

对每个分析 Stage，统计：
- 有数据分区的 Task 耗时范围
- 空分区（极短耗时，< 100 ms）Task 数量
- 最长 / 最短耗时比（> 3x 视为值得关注，> 10x 视为严重倾斜）

---

## 第七步：异常与警告扫描

```bash
# 严重错误（OOM、Executor lost、fetchFailed 等）
grep -n "ERROR\|Exception\|OOM\|OutOfMemory\|Executor lost\|fetchFailed\|Connection refused" "$BASE" \
  | grep -v "deprecated\|HiveConf\|Hive Session\|mapred-default\|METASTORE\|Ranger\|ranger" \
  | head -30

# 废弃配置警告
grep -n "deprecated\|has been deprecated" "$BASE" | grep -oP "'[^']+' has been deprecated" | sort -u

# 重要 RuntimeWarning
grep -n "RuntimeWarning\|Failed to add file\|pyFiles" "$BASE" | head -5

# NameNode HA 异常
grep -n "standby\|Operation category READ" "$BASE" | head -3
```

按严重程度分级：CRITICAL / HIGH / MEDIUM / LOW。

---

## 第八步：输出结构化报告

在对话中输出完整分析报告，格式如下：

```
## Spark 任务分析报告

### 基本信息
- Application ID：
- 任务名称：
- 运行模式：
- 时间范围：开始 ~ 结束（总耗时 X 秒）
- 提交用户：

### 执行概览
- Job：总计 X 个（成功 X / 失败 X）
- Stage：总计 X 个实际执行（Stage ID 跨度 X，其余命中缓存复用）
- Task：总计 X 个（失败重试 X 次）
- 耗时 Top 3 Stage：...

### 资源配置
- Executor 内存：Xg | Executor 数量：X 个 | CPU Cores：X Core/个
- 集群总并行度：X | 动态资源分配：开启/关闭
- MemoryStore：X MiB/Executor | GC 策略：...

### 数据处理量
- 输入：X 个文件 | 初始并行度：X 分区 | 主处理并行度：X 分区
- Shuffle Read/Write：X | Spill to Disk：X

### Executor 节点参与情况与使用率
（完整表格：节点映射 + Task 分配统计 + 冷启动耗时）

### 数据倾斜分析
（各关键 Stage 的 Task 耗时分布）

### 异常与警告
（按严重程度排序）

### GC 状况
- GC 总耗时：Xs（占总耗时 X%）
- Full GC 次数：X | 评估：正常/需关注/严重
```

信息缺失时标注"未找到"，不猜测。

---

## 第九步：生成 Markdown 文档

将报告写入日志根目录：

```
<LOG_ROOT>/spark-analysis-report.md
```

文件名固定为 `spark-analysis-report.md`。若已存在，覆盖写入。

报告结构与对话输出一致，额外包含：
- 各 Stage 完整耗时列表
- 各 Job GC 耗时明细表
- 日志文件清单附录（路径 / 行数 / 大小 / 说明）

---

## 注意事项

- **大文件限制**：单文件超过 256 KB 时只能用 grep/sed，不能整体 Read
- **并行提取**：对多个 Executor 日志的相同查询，在同一 Bash 命令中用 for 循环批量处理
- **AM vs Driver**：container_000001 在 executor/ 目录下通常是 AM + SparkContext，不是纯 Executor
- **Stage 缓存复用**：某些 Stage ID 出现在 `Parents of final stage` 但从未出现在 `finished in` 中，说明该 Stage 被复用，不计入实际执行数
- **Task 总数统计**：汇总所有 `Submitting X missing tasks` 中的 X，不要数 `Finished task` 行（AM 日志中仅记录部分）
