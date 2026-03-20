---
name: spark-tool-transformer
description: 将本地 Python 数据处理工具改造为 Spark 分布式任务的专项指南。当用户需要以下操作时立即触发：(1) 将依赖本地文件系统的 Python 工具迁移到 Spark/YARN 集群；(2) 让 Python 脚本支持 OBS（对象存储）路径读写；(3) 将串行文件处理循环改造为 Spark 并行化；(4) 用户提到"改造成 Spark 任务"、"分布式处理"、"YARN 集群"、"OBS 路径"等关键词；(5) 用户有使用 os.walk / os.getenv / os.path 的本地工具，想在云上 Spark 环境运行。即使用户只是问"怎么让这个脚本在 Spark 上跑"，也应立即触发本 skill。
---

# Spark Tool Transformer

将本地 Python 数据处理工具改造为在 Spark（YARN）集群上运行、读写 OBS 路径的分布式任务。

## 核心原则

- **Spark 调优参数不进代码**：executor-cores、executor-memory 等由集群环境统一管理
- **保持向后兼容**：改造时新增文件、新增参数（默认回退），不破坏原有代码可用性
- **Executor 内不能用 SparkContext**：SparkContext 不可序列化，不能在 executor 函数中引用

---

## 改造产物：新建三个文件

### 文件一：`spark_env_config.py`

负责从 `spark.sparkContext.environment` 读取所有配置。不用 `os.getenv`，因为 Spark 通过 `--conf spark.executorEnv.XXX` / `--conf spark.yarn.appMasterEnv.XXX` 传递的环境变量只能通过这个接口拿到。

```python
# -*- coding: utf-8 -*-
import json
import logging
from dataclasses import dataclass, field
from typing import Dict

from pyspark.sql import SparkSession

logger = logging.getLogger(__name__)


@dataclass
class AppConfig:
    """
    Spark 调优参数（executor-cores / executor-memory 等）由集群环境统一管理，不在此处配置
    """
    input_path: str
    output_path: str
    num_partitions: int = 30
    # 按业务需要添加其他字段，例如：
    # doc_id: str = ""
    # topic_select: list = field(default_factory=list)

    def validate(self) -> None:
        if not self.input_path or not self.input_path.strip():
            raise ValueError("input_path is empty")
        if not self.output_path or not self.output_path.strip():
            raise ValueError("output_path is empty")
        logger.info("AppConfig validation passed")


class ConfigLoader:
    """
    环境变量约定：
      PLUGIN_TOOL_INPUT  → JSON 字符串 {"input_dir": "obs://bucket/path/"}
      PLUGIN_TOOL_OUTPUT → JSON 字符串 {"output_dir": "obs://bucket/path/"}
      NUM_PARTITIONS     → 整数字符串，默认 "30"
    """

    ENV_MAPPING: Dict[str, str] = {
        "input_path":    "PLUGIN_TOOL_INPUT",
        "output_path":   "PLUGIN_TOOL_OUTPUT",
        "num_partitions": "NUM_PARTITIONS",
        # 业务字段示例："doc_id": "DOC_ID"
    }

    @classmethod
    def from_spark_env(cls, spark: SparkSession) -> AppConfig:
        env_vars = dict(spark.sparkContext.environment)

        input_raw  = env_vars.get(cls.ENV_MAPPING["input_path"],  "{}")
        output_raw = env_vars.get(cls.ENV_MAPPING["output_path"], "{}")
        num_str    = env_vars.get(cls.ENV_MAPPING["num_partitions"], "30")

        try:
            input_path  = json.loads(input_raw).get("input_dir",  "")
            output_path = json.loads(output_raw).get("output_dir", "")
        except json.JSONDecodeError:
            logger.error(f"Failed to parse path JSON: input={input_raw}, output={output_raw}")
            input_path = output_path = ""

        return AppConfig(
            input_path=input_path,
            output_path=output_path,
            num_partitions=int(num_str) if num_str else 30,
        )
```

**有业务字段时**（如 `doc_id`、`topic_select`）：在 `AppConfig` 添加字段，在 `ENV_MAPPING` 添加映射，在 `from_spark_env` 添加解析逻辑。`topic_select` 这类 JSON 数组字段用 `json.loads` 解析，普通字符串字段直接 `env_vars.get`。

---

### 文件二：`spark_file_loader.py`

继承原有 `FileLoader`，用 Hadoop FileSystem API 替代 `os.walk`，支持 OBS 路径。

```python
# -*- coding: utf-8 -*-
import logging
from typing import Optional

from pyspark.rdd import RDD
from pyspark.sql import DataFrame, SparkSession

from <your_package>.file_loader import FileLoader
from <your_package>.spark_env_config import AppConfig, ConfigLoader

logger = logging.getLogger(__name__)


class SparkFileLoader(FileLoader):
    """
    OBS 路径格式：obs://bucket-name/path/to/dir/
    前提：集群已部署 hadoop-huaweicloud jar，
         OBS 凭证（fs.obs.access.key / fs.obs.secret.key / fs.obs.endpoint）由集群配置注入
    """

    def __init__(self, app_name: str = "SparkFileLoader"):
        self._spark: SparkSession = SparkSession.builder.appName(app_name).getOrCreate()
        config: AppConfig = ConfigLoader.from_spark_env(self._spark)
        config.validate()
        super().__init__(config.input_path)
        self.output_path:    str = config.output_path
        self.num_partitions: int = config.num_partitions

    def _get_hadoop_fs(self, path: str):
        sc   = self._spark.sparkContext
        conf = sc._jsc.hadoopConfiguration()
        uri  = sc._jvm.java.net.URI.create(path)
        return sc._jvm.org.apache.hadoop.fs.FileSystem.get(uri, conf)

    # ---- 覆写：os.walk → Hadoop FS 递归列举 ----

    def collect_all_file_from_specific_path(self, start_path: str, file_type: str) -> list:
        suffix      = self._generate_file_suffix(file_type)
        sc          = self._spark.sparkContext
        fs          = self._get_hadoop_fs(start_path)
        hadoop_path = sc._jvm.org.apache.hadoop.fs.Path(start_path)
        file_iter   = fs.listFiles(hadoop_path, True)   # recursive=True

        res = []
        while file_iter.hasNext():
            p = file_iter.next().getPath().toString()
            if p.endswith(suffix):
                res.append(p)
        logger.info(f"Found {len(res)} {file_type} files under {start_path}")
        return res

    # ---- 新增：并行读取 ----

    def load_files_as_rdd(self, file_type: str) -> RDD:
        """返回 OBS 文件路径组成的 RDD，供后续 map/flatMap 处理"""
        paths = self.collect_all_file_path(file_type)
        if not paths:
            return self._spark.sparkContext.emptyRDD()
        return self._spark.sparkContext.parallelize(paths, self.num_partitions)

    def load_json_as_dataframe(self) -> DataFrame:
        return self._spark.read.json(self.input_path)

    # ---- 新增：分布式写出 ----

    def write_rdd(self, rdd: RDD, output_path: Optional[str] = None) -> None:
        (output_path or self.output_path) and rdd.saveAsTextFile(output_path or self.output_path)

    def write_dataframe(self, df: DataFrame, output_path: Optional[str] = None) -> None:
        df.write.mode("overwrite").json(output_path or self.output_path)
```

---

### 文件三：改写主入口

```python
# -*- coding: utf-8 -*-
import json, logging, os, shutil, subprocess, tempfile

from pyspark.sql import SparkSession

from <your_package>.spark_env_config import AppConfig, ConfigLoader
from <your_package>.spark_file_loader import SparkFileLoader

logger = logging.getLogger(__name__)


# ── Executor 函数（运行在 worker 节点，不能引用 SparkContext）──────────────

def process_file_bytes(path_bytes: tuple, config_dict: dict) -> list:
    """
    path_bytes: sc.binaryFiles() 产出的 (obs_path, file_bytes) 元组
    返回 list of (relative_output_path, data_dict)
    relative_output_path 由调用方拼接 OBS output_path 前缀
    """
    obs_path, file_bytes = path_bytes
    work_dir = tempfile.mkdtemp()
    try:
        # 1. 将 sc.binaryFiles() 读取的字节写入 executor 本地临时文件
        #    ⚠️  不要用 subprocess hadoop fs -get：executor 的 environment.zip
        #         里 hadoop 不在 PATH，会直接 FileNotFoundError
        local_input = os.path.join(work_dir, "input")
        os.makedirs(local_input)
        local_file = os.path.join(local_input, os.path.basename(obs_path))
        with open(local_file, "wb") as f:
            f.write(file_bytes)

        # 2. 调用现有处理逻辑（传入 base_dir 解耦路径依赖）
        results = your_processing_function(local_file, base_dir=work_dir, **config_dict)
        return results

    except Exception as e:
        # ⚠️  必须 raise，不能 return []
        #     executor 的 logger 输出不会出现在 driver 日志里，
        #     静默 return [] 会让所有错误完全不可见，表现为结果全部为 0
        logger.error(f"Failed: {obs_path}: {e}", exc_info=True)
        raise
    finally:
        shutil.rmtree(work_dir, ignore_errors=True)


# ── Driver 端写 OBS（通过 Hadoop FS API，支持任意 JSON）─────────────────────

def write_json_to_obs(data: dict, obs_path: str, spark: SparkSession) -> None:
    sc   = spark.sparkContext
    fs   = sc._jvm.org.apache.hadoop.fs.FileSystem.get(
               sc._jvm.java.net.URI.create(obs_path),
               sc._jsc.hadoopConfiguration())
    path = sc._jvm.org.apache.hadoop.fs.Path(obs_path)
    out  = fs.create(path, True)   # overwrite=True
    try:
        out.write(json.dumps(data, ensure_ascii=False, indent=4).encode("utf-8"))
    finally:
        out.close()


# ── 主入口 ────────────────────────────────────────────────────────────────────

def main():
    spark  = SparkSession.builder.appName("YourSparkTool").getOrCreate()
    config = ConfigLoader.from_spark_env(spark)
    config.validate()

    loader     = SparkFileLoader()
    file_paths = loader.collect_all_file_path("your_file_type")   # 替换为实际类型

    if not file_paths:
        logger.warning("No files found, exiting")
        return

    config_dict = {
        # 传给 executor 的业务配置，不含 SparkContext
        # "doc_id": config.doc_id,
    }

    # sc.binaryFiles() 通过 Spark 原生 OBS connector 读取文件字节并分发给 executor，
    # 无需 executor 环境里有 hadoop 命令
    all_results: list = (
        spark.sparkContext
        .binaryFiles(",".join(file_paths), minPartitions=config.num_partitions)
        .flatMap(lambda p: process_file_bytes(p, config_dict))
        .collect()
    )

    logger.info(f"Total results: {len(all_results)}")

    output_base = config.output_path.rstrip("/")
    success = failed = 0
    for rel_path, data in all_results:
        try:
            write_json_to_obs(data, f"{output_base}/{rel_path}", spark)
            success += 1
        except Exception as e:
            logger.error(f"Write failed {rel_path}: {e}")
            failed += 1

    logger.info(f"Done | success={success} | failed={failed}")


if __name__ == "__main__":
    main()
```

---

## 改造现有文件的两个关键模式

### 模式一：解耦路径依赖（`os.getcwd()` → `base_dir` 参数）

Spark executor 的工作目录不固定，不能依赖 `os.getcwd()` 构建路径。改法：为相关函数增加 `base_dir=None` 参数，`None` 时回退原行为，保持向后兼容。

```python
# 改造前
def process(name):
    path = os.path.join(os.getcwd(), f"temp/{name}/data")

# 改造后（向后兼容）
def process(name, base_dir=None):
    base = base_dir or os.getcwd()
    path = os.path.join(base, f"temp/{name}/data")
```

调用处同步传入 executor 的工作目录：

```python
# executor 函数内
work_dir = tempfile.mkdtemp()
process(name, base_dir=work_dir)
```

### 模式二：数据收集模式（写文件 → 返回数据）

Executor 无法直接写 OBS（不能用 `sc._jvm`），改为返回数据，由 Driver 统一写出。

```python
# 改造前
def process_folder(folder, output_dir):
    for file in os.listdir(folder):
        result = parse(file)
        FileLoader.write_file(result, os.path.join(output_dir, file + ".json"))

# 改造后（保留原函数，新增 collect 版本）
def collect_results(folder) -> list:
    """返回 (relative_output_path, data_dict) 列表"""
    results = []
    for file in os.listdir(folder):
        result = parse(file)
        if result:
            results.append((f"{folder}/{file}.json", result))
    return results
```

---

## 改造检查清单

分析现有代码时，逐项检查：

| 项目 | 原实现 | Spark 改法 |
|---|---|---|
| 环境变量 | `os.getenv("KEY")` | `spark.sparkContext.environment.get("KEY")` |
| 输入路径格式 | 纯字符串 | JSON `{"input_dir": "obs://..."}` |
| 文件遍历 | `os.walk(path)` | Hadoop FS `listFiles(path, recursive=True)` |
| 路径拼接 | `os.path.join(os.getcwd(), ...)` | `os.path.join(base_dir, ...)` |
| 处理入口 | `for f in files: process(f)` | `sc.parallelize(files).flatMap(process_fn)` |
| 写本地文件 | `open(path, 'w')` / `write_file` | 返回数据 → Driver 用 Hadoop FS 写 OBS |
| 读 OBS 文件（executor） | `open(local_path)` | `sc.binaryFiles(paths)` 读取字节，executor 收到后写本地再读；**不要用 `subprocess hadoop fs -get`**，executor 的 PATH 里没有 hadoop |

---

## 注意事项

- `sc._jvm` / `sc._jsc` 只能在 **Driver** 上使用，不能在 executor 函数内使用
- executor 函数必须是可序列化的顶层函数或 lambda，不能引用 SparkContext、SparkSession
- executor 的本地临时目录用 `tempfile.mkdtemp()`，处理完毕后 `shutil.rmtree` 清理
- `.collect()` 将所有结果拉回 Driver，适合结果总量可控的场景（通常 < 1GB）；结果量大时改用 `write_dataframe` 直接写 OBS
- **Spark 并行粒度是文件级**：`sc.binaryFiles(paths)` 每个文件对应一个 partition，单文件时 Spark 没有提效，只有调度开销；多文件才有并行收益

---

## 常见陷阱（来自实际调试经验）

### ⚠️ 陷阱一：executor 里用 subprocess 调 hadoop 命令

```python
# ❌ 错误写法 — executor 的 environment.zip 里 hadoop 不在 PATH
subprocess.run(["hadoop", "fs", "-get", obs_path, local_file], check=True)
# → FileNotFoundError: [Errno 2] No such file or directory: 'hadoop'

# ✅ 正确写法 — 用 sc.binaryFiles() 在 Driver 读取文件字节，executor 直接写本地
# 见"文件三"中的 process_file_bytes 模板
```

### ⚠️ 陷阱二：executor 里静默吞掉异常

```python
# ❌ 错误写法 — executor 的日志不会出现在 driver 输出里，return [] 让所有错误消失
except Exception as e:
    logger.error(f"Failed: {e}")
    return []   # 表面上任务成功，实际结果全为 0，极难排查

# ✅ 正确写法 — raise 让 Spark task 真正失败，driver 日志可见完整 traceback
except Exception as e:
    logger.error(f"Failed: {e}", exc_info=True)
    raise
```

### ⚠️ 陷阱三：多文档类型共用含类型特定校验的解析函数

当工具需要处理多种格式（如 hwics / hdx）时，如果共用一个解析函数，该函数内部包含某种格式特有的字段校验（如检查 `DC.Identifier` meta 标签是否存在），其他格式的文件会被静默丢弃。

```python
# ❌ 错误写法 — convert() 里有 hwics 特有的 DC.Identifier 校验
#    hdx 文件没有这个字段 → convert() 返回 None → 文件被跳过 → 0 结果
result, title = convert(html_path, doc_id, topic_bookmap)  # hdx 全部返回 None

# ✅ 正确写法 — 抽取纯内容解析函数，URI 构建由各类型自己负责
def parse_html(html_path) -> tuple[str, str | None]:
    """只提取正文和标题，不涉及任何 URI 或格式特定校验"""
    ...
    return text_content, title

# hdx 处理函数调用 parse_html，URI 由 navi.xml 的 topic_id 构建
text, title = parse_html(html_path)
result = {"uri": f"...hdx...?id={topic_id}", "text": text, "title": title}
```
