# LLaMA-Factory 数据 README 解读

本文是对 `data/README.md` 的中文梳理，重点解释 LLaMA-Factory 如何注册、识别和读取训练数据集。

## 核心作用

`data/README.md` 主要说明一件事：

> 不能只把一个 `json`、`jsonl`、`csv` 等数据文件放进 `data/` 目录就直接训练。你需要先在 `dataset_info.json` 中注册这个数据集，告诉 LLaMA-Factory 数据文件在哪里、是什么格式，以及哪些字段代表 prompt、response、chosen、rejected、images 等。

默认数据目录是：

```text
./data
```

默认数据集注册文件是：

```text
data/dataset_info.json
```

训练配置中通常会写：

```yaml
dataset: dataset_name
```

这里的 `dataset_name` 必须能在 `dataset_info.json` 中找到。

## dataset_info.json 的作用

`dataset_info.json` 是数据集登记表。每个数据集都要有一个描述，例如：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "formatting": "alpaca",
    "ranking": false,
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output"
    }
  }
}
```

其中：

| 字段 | 含义 |
| --- | --- |
| `dataset_name` | 训练参数中 `dataset: xxx` 使用的名字 |
| `file_name` | 本地数据文件或目录名 |
| `formatting` | 数据格式，支持 `alpaca` 或 `sharegpt` |
| `ranking` | 是否是偏好数据集 |
| `columns` | 把原始数据字段映射到 LLaMA-Factory 内部字段 |
| `tags` | ShareGPT/OpenAI 风格对话中的角色字段映射 |

`columns` 的左边是 LLaMA-Factory 内部使用的标准字段，右边是你自己的数据文件里的真实字段名。

例如你的数据长这样：

```json
{
  "question": "你好",
  "answer": "你好，有什么可以帮你？"
}
```

那么可以这样注册：

```json
{
  "my_dataset": {
    "file_name": "my_data.json",
    "columns": {
      "prompt": "question",
      "response": "answer"
    }
  }
}
```

## 数据来源

README 支持多种数据来源：

| 字段 | 来源 |
| --- | --- |
| `hf_hub_url` | Hugging Face Hub 数据集 |
| `ms_hub_url` | ModelScope 数据集 |
| `script_url` | 自定义数据加载脚本 |
| `cloud_file_name` | S3/GCS 等云存储文件 |
| `file_name` | 本地 `data/` 目录中的文件或目录 |

如果指定了更高优先级的数据来源，例如 `hf_hub_url`，则会忽略本地 `file_name`。

本地文件类型支持：

```text
json, jsonl, csv, parquet, arrow
```

## 支持的两类主要格式

LLaMA-Factory 主要支持两类数据格式：

```text
alpaca
sharegpt
```

区别如下：

| 格式 | 适用场景 |
| --- | --- |
| `alpaca` | 简单的 instruction/input/output 格式 |
| `sharegpt` | 多轮对话、多角色、工具调用、多模态数据 |

## Alpaca 格式

Alpaca 格式适合普通 SFT 数据。典型结构如下：

```json
[
  {
    "instruction": "user instruction",
    "input": "user input",
    "output": "model response",
    "system": "system prompt",
    "history": [
      ["old user question", "old model answer"]
    ]
  }
]
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `instruction` | 用户指令，必需 |
| `input` | 用户输入，可选 |
| `output` | 模型回答，必需 |
| `system` | system prompt，可选 |
| `history` | 历史对话，可选 |

训练时，用户输入会由下面两部分拼接：

```text
instruction + "\n" + input
```

需要注意：`history` 中的 assistant 回复在 SFT 中也会参与学习，也会计算 loss。

对应的 `dataset_info.json` 配置示例：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output",
      "system": "system",
      "history": "history"
    }
  }
}
```

### 推理模型和 CoT

如果模型有 reasoning 能力，例如 Qwen3，而数据中没有 chain-of-thought，LLaMA-Factory 可能会自动补空 CoT。

`enable_thinking` 会影响这个空 CoT 放在哪里：

| 参数状态 | 行为 |
| --- | --- |
| `True` | 空 CoT 放入模型回复，参与 loss |
| `False` | 空 CoT 放入用户 prompt，不参与 loss |
| `None` | 混合处理，有复杂性，需谨慎使用 |

训练和推理时应保持 `enable_thinking` 配置一致。

## 预训练数据

预训练数据只使用 `text` 字段：

```json
[
  {"text": "document"},
  {"text": "document"}
]
```

注册配置：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "columns": {
      "prompt": "text"
    }
  }
}
```

这类数据不是问答训练，而是继续学习普通文本。

## 偏好数据集

偏好数据集用于：

```text
Reward Modeling, DPO, ORPO, SimPO
```

这类数据需要一好一坏两个回答：

```json
[
  {
    "instruction": "user instruction",
    "input": "user input",
    "chosen": "chosen answer",
    "rejected": "rejected answer"
  }
]
```

注册时必须设置：

```json
"ranking": true
```

示例：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "ranking": true,
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "chosen": "chosen",
      "rejected": "rejected"
    }
  }
}
```

## ShareGPT 格式

ShareGPT 格式适合更复杂的对话数据。核心字段通常是 `conversations`：

```json
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "user instruction"
      },
      {
        "from": "gpt",
        "value": "model response"
      }
    ],
    "system": "system prompt",
    "tools": "tool description"
  }
]
```

支持的角色包括：

| 角色 | 含义 |
| --- | --- |
| `human` | 用户 |
| `gpt` | assistant |
| `observation` | 工具返回结果 |
| `function_call` | 工具调用参数 |
| `system` | system prompt |

README 中要求角色顺序要合理：

```text
human / observation 应出现在用户侧位置
gpt / function_call 应出现在模型侧位置
```

模型会学习 `gpt` 和 `function_call` 这些模型侧内容。

注册示例：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "formatting": "sharegpt",
    "columns": {
      "messages": "conversations",
      "system": "system",
      "tools": "tools"
    }
  }
}
```

## tags 的作用

`tags` 用于 ShareGPT 格式，告诉系统一条消息里哪个字段表示角色，哪个字段表示内容。

默认 ShareGPT 消息长这样：

```json
{
  "from": "human",
  "value": "hello"
}
```

默认映射相当于：

```json
{
  "role_tag": "from",
  "content_tag": "value",
  "user_tag": "human",
  "assistant_tag": "gpt"
}
```

如果你的数据是 OpenAI 风格：

```json
{
  "role": "user",
  "content": "hello"
}
```

就需要配置：

```json
{
  "tags": {
    "role_tag": "role",
    "content_tag": "content",
    "user_tag": "user",
    "assistant_tag": "assistant",
    "system_tag": "system"
  }
}
```

## KTO 数据集

KTO 数据需要额外字段 `kto_tag`，表示人类反馈的布尔标签：

```json
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "user instruction"
      },
      {
        "from": "gpt",
        "value": "model response"
      }
    ],
    "kto_tag": true
  }
]
```

注册示例：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "formatting": "sharegpt",
    "columns": {
      "messages": "conversations",
      "kto_tag": "kto_tag"
    }
  }
}
```

## 多模态数据

多模态数据基于 ShareGPT 格式，并额外增加：

```text
images, videos, audios
```

图片数据示例：

```json
[
  {
    "conversations": [
      {
        "from": "human",
        "value": "<image>user instruction"
      },
      {
        "from": "gpt",
        "value": "model response"
      }
    ],
    "images": [
      "image path"
    ]
  }
]
```

关键规则：

| 模态 | 占位符 | 路径字段 |
| --- | --- | --- |
| 图片 | `<image>` | `images` |
| 视频 | `<video>` | `videos` |
| 音频 | `<audio>` | `audios` |

占位符数量必须和对应路径列表数量一致。

例如：

```text
一个样本里有 2 个 <image>，images 列表里也必须有 2 个图片路径。
```

## OpenAI 格式

OpenAI 格式可以看作 ShareGPT 格式的特殊情况。

示例：

```json
[
  {
    "messages": [
      {
        "role": "system",
        "content": "system prompt"
      },
      {
        "role": "user",
        "content": "user instruction"
      },
      {
        "role": "assistant",
        "content": "model response"
      }
    ]
  }
]
```

注册时仍然写：

```json
"formatting": "sharegpt"
```

但要通过 `tags` 映射 OpenAI 的字段：

```json
{
  "dataset_name": {
    "file_name": "data.json",
    "formatting": "sharegpt",
    "columns": {
      "messages": "messages"
    },
    "tags": {
      "role_tag": "role",
      "content_tag": "content",
      "user_tag": "user",
      "assistant_tag": "assistant",
      "system_tag": "system"
    }
  }
}
```

## 总结

`data/README.md` 可以理解为 LLaMA-Factory 的数据接入说明书：

1. 所有数据集都要先注册到 `dataset_info.json`。
2. `dataset_info.json` 告诉程序数据文件在哪里、格式是什么、字段怎么映射。
3. 普通 SFT 数据通常用 `alpaca` 或 `sharegpt`。
4. 偏好数据需要 `ranking: true`，并提供 `chosen` 和 `rejected`。
5. ShareGPT 格式通过 `messages`/`conversations` 表示多轮对话。
6. `tags` 用来适配不同消息字段命名，例如 OpenAI 格式的 `role` 和 `content`。
7. 多模态数据需要 `images`、`videos` 或 `audios` 字段，并与 `<image>`、`<video>`、`<audio>` 占位符数量对应。

最实用的理解是：

> `data/README.md` 不是训练脚本说明，而是告诉你如何把自己的数据整理成 LLaMA-Factory 能识别的格式。
