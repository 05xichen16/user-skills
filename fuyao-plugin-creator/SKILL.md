---
name: fuyao-plugin-creator
description: >
  扶摇平台插件代码生成器。当用户需要为扶摇（Fuyao）平台开发任意类型的插件时，
  使用此 skill 自动生成符合规范的完整插件代码，包括目录结构、DSL 配置文件
  actions.yaml、主脚本 main.py 以及 README.md。插件类型涵盖数据采集、数据清洗、
  数据处理、数据发布等所有扶摇平台插件。当用户提到"扶摇插件"、"扶摇工具"、
  "actions.yaml"、"PLUGIN_INPUT_FILE_PATH"、"扶摇平台开发"、"清洗插件"、
  "采集插件"、"处理插件"、"发布插件"等关键词时，立即触发此 skill，哪怕用户
  没有明确说"生成插件代码"也应主动使用。
---

# 扶摇平台插件生成器

你的任务是根据用户的功能描述，生成一套完整的、符合扶摇平台规范的插件代码。

## 工作流程

1. **理解需求**：从用户描述中提取插件功能、处理逻辑、所需输入参数
2. **推断分类与命名**：根据功能自动确定插件类别和名称
3. **生成完整代码**：输出目录结构、actions.yaml、main.py、README.md
4. **说明关键设计**：简要解释参数配置和处理逻辑

---

## 插件分类与命名规则

| 分类 | 关键词（用于目录/name 命名） | 示例 |
|------|---------------------------|------|
| 数据采集 | `collector` / `crawler` / `fetcher` / `importer` | `Web_Data_Collector` |
| 数据清洗 | `clean` / `filter` / `dedup` / `deduplication` / `remove` / `normalize` | `Garbled_Char_Filter` |
| 数据处理 | `processor` / `converter` / `transformer` / `analyzer` | `Text_Format_Converter` |
| 数据发布 | `publisher` / `exporter` / `pusher` / `uploader` | `Dataset_Publisher` |

**目录命名**：大写首字母驼峰 + 下划线，如 `My_Plugin_Name`
**插件 name 字段**：小写 + 下划线，如 `my_plugin_name`
**参数名**：全大写 + 下划线，如 `COL_LIST`、`THRES`、`TARGET_URL`

---

## 输出格式

按以下顺序输出，每个文件用标题分隔：

### 1. 目录结构

```
PluginName/
├── dsl/
│   └── actions.yaml
├── script/
│   └── main.py
├── test/
│   └── test_main.py
└── README.md
```

### 2. `dsl/actions.yaml`

严格遵循以下结构（必填字段一个不能少）：

```yaml
name: plugin_name
description: 插件功能的一句话描述
instruction: https://wiki-url  # 若无则省略此行
common_input_file_path:
  PLUGIN_INPUT_FILE_PATH:
    default: ''
    description: 公共输入文件路径，固定名字，需要输入的文件路径，插件可以获取到一个文件目录
    required: true
inputs:
  PARAM_NAME:
    description: 参数说明
    type: string        # string | float | int | list | json
    label: 参数显示标签
    input_type: text    # text | radio | fieldselect | jsonlist
    options: null       # radio 时填选项列表，否则为 null
    required: true
    default: ''
outputs:
  OUT_PUT_PATH:
    description: 结果文件输出路径
    type: string
    label: 结果文件路径
    input_type: text
    options: null
    required: true
runs:
  using: python PluginName/script/main.py
```

**参数类型 → 控件类型对应关系**：

| 参数类型 | input_type | 说明 |
|----------|-----------|------|
| `string` / `float` / `int` | `text` | 普通文本输入 |
| `string`（布尔/枚举） | `radio` | 需填写 `options` 列表 |
| `list` | `fieldselect` | 多字段选择 |
| `json` | `jsonlist` | 复杂结构，需加 `model` 字段定义子字段 |

**jsonlist model 示例**：
```yaml
  CLEAN_INFO:
    type: json
    input_type: jsonlist
    model:
      - key: KEY
        label: 字段名
        description: 要匹配的字段名
        required: true
      - key: VALUE
        label: 字段值
        description: 要匹配的字段值
        required: true
```

### 3. `script/main.py`

必须包含以下所有元素：

**文件头**（逐字复制，日期填写实际日期）：
```python
#!/usr/bin/python3.7
# -*- coding: utf-8 -*-
# Copyright (c) Huawei Technologies Co., Ltd. 2023-2023. All rights reserved.
# @Time: YYYY/MM/DD
```

**导入与路径配置**：
```python
import os
import sys
import logging
import json
from pathlib import Path

logging.basicConfig(format='%(asctime)s - %(levelname)s - %(message)s', level=logging.INFO)

project_root = os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
if project_root not in sys.path:
    sys.path.insert(0, project_root)

from common_tools.json_tools import get_json_files, json_path_load, json_and_jsonl_save
from common_tools.file_tools import get_file_path_of_output_dir
```

**参数获取**（在 `__main__` 块中）：
```python
if __name__ == "__main__":
    input_dir = os.getenv("PLUGIN_INPUT_FILE_PATH")
    output_dir = os.getenv("OUT_PUT_PATH")
    param = os.getenv("PARAM_NAME")
    # list/json 类型参数需解析：
    list_param = json.loads(os.getenv("LIST_PARAM", "[]"))
    json_param = json.loads(os.getenv("JSON_PARAM", "[]"))

    if not input_dir or not output_dir:
        logging.error("缺少必需的环境变量: PLUGIN_INPUT_FILE_PATH 或 OUT_PUT_PATH")
        sys.exit(1)

    main(input_dir, output_dir, ...)
```

**函数规范**：
- `main()` 和所有业务函数必须有 docstring
- 文件处理循环中每个文件用 `try/except Exception` 包裹，异常时 `logging.error` + `continue`
- 使用 `get_file_path_of_output_dir(input_file, input_dir, output_dir)` 获取输出路径
- 大文件推荐使用 `stream_read` 读取、`JsonStreamWriter` 写入

### 4. `README.md`

```markdown
# PluginName

## 功能描述
...

## 参数说明
| 参数名 | 类型 | 默认值 | 是否必填 | 说明 |
|--------|------|--------|----------|------|
| PARAM1 | string | - | 是 | 说明 |

## 使用示例
...

## 输入输出示例
**输入**
{"field": "value"}

**输出**
{"field": "processed_value"}
```

---

## 生成质量自查清单

生成完代码后逐项检查：

- [ ] 目录名：大写首字母驼峰+下划线，体现插件类别关键词
- [ ] `actions.yaml`：包含 name、description、common_input_file_path、inputs、outputs、runs 全部必填字段
- [ ] 固定字段名未被修改：`PLUGIN_INPUT_FILE_PATH`、`OUT_PUT_PATH`
- [ ] 参数名全大写+下划线
- [ ] `runs.using` 路径指向正确插件目录和脚本文件
- [ ] `main.py` 含文件头版权注释（#!/usr/bin/python3.7 + Copyright）
- [ ] 所有参数通过 `os.getenv` 获取；list/json 类型用 `json.loads` 解析
- [ ] 含必需参数空值校验（`if not input_dir or not output_dir`）
- [ ] 文件处理有 `try/except` 异常处理
- [ ] 所有函数有 docstring

---

## 需要更多信息时

描述足够清晰时直接生成，无需额外询问。描述不完整时，优先确认：

- 插件处理的核心数据字段是哪些？
- 是否有可配置的阈值/规则参数？
- 过滤/处理的方向（保留 vs 删除符合条件的数据）？
- 是否有特殊算法或外部依赖？
