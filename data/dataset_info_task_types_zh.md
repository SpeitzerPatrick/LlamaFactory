# dataset_info.json 中的数据集类型说明

`data/dataset_info.json` 不是一个训练样本文件，而是 LLaMA-Factory 的数据集注册表。它登记了项目支持的各种数据集，并说明每个数据集从哪里读取、使用什么格式、字段如何映射。

不能简单理解为：

```text
dataset_info.json 里的数据集全都是对话类任务
```

更准确的理解是：

```text
dataset_info.json 登记的是 LLaMA-Factory 支持的数据集清单。
其中很多是对话/指令微调数据，但也包括预训练文本、偏好排序、多模态、KTO、工具调用等数据。
```

## 1. 对话/指令类 SFT 数据

这类数据最接近“用户输入 -> 模型输出”。

Alpaca 风格示例：

```json
{
  "instruction": "Are you created by Google?",
  "input": "",
  "output": "No, I am {{name}}, an AI assistant developed by {{author}}."
}
```

这条样本可以理解为：

```text
用户：Are you created by Google?
助手：No, I am {{name}}, an AI assistant developed by {{author}}.
```

ShareGPT 风格示例：

```json
{
  "conversations": [
    {
      "from": "human",
      "value": "Hello"
    },
    {
      "from": "gpt",
      "value": "Hi"
    }
  ]
}
```

代表数据集包括：

```text
alpaca_en
alpaca_zh
alpaca_gpt4_en
alpaca_gpt4_zh
guanaco
belle_2m
belle_1m
belle_0.5m
belle_dialog
belle_math
sharegpt4
ultrachat_200k
lmsys_chat
evol_instruct
```

这是 `dataset_info.json` 中的主体类型。

## 2. 预训练/续训文本数据

这类数据不是对话，也不是“用户问、助手答”。它是纯文本语料，用于 continued pretraining，让模型继续学习语言分布、领域知识或代码分布。

示例：

```json
{
  "text": "Large language models are trained on large corpora..."
}
```

这类数据通常只有文本字段映射，例如：

```json
{
  "columns": {
    "prompt": "text"
  }
}
```

或者：

```json
{
  "columns": {
    "prompt": "content"
  }
}
```

代表数据集包括：

```text
wiki_demo
c4_demo
refinedweb
redpajama_v2
wikipedia_en
wikipedia_zh
pile
skypile
fineweb
fineweb_edu
the_stack
starcoder_python
```

这些数据不应该理解成对话任务。

## 3. 偏好排序数据

偏好排序数据用于 DPO、Reward Modeling、ORPO、SimPO 等训练。

这类数据不是“一个输入对应一个标准输出”，而是：

```text
同一个问题下，比较哪个回答更好。
```

Alpaca 风格示例：

```json
{
  "instruction": "user question",
  "chosen": "better answer",
  "rejected": "worse answer"
}
```

ShareGPT 风格示例：

```json
{
  "conversations": [
    {
      "from": "human",
      "value": "user question"
    }
  ],
  "chosen": {
    "from": "gpt",
    "value": "better answer"
  },
  "rejected": {
    "from": "gpt",
    "value": "worse answer"
  }
}
```

在 `dataset_info.json` 中，这类数据通常有：

```json
{
  "ranking": true
}
```

代表数据集包括：

```text
dpo_en_demo
dpo_zh_demo
dpo_mix_en
dpo_mix_zh
ultrafeedback
coig_p
rlhf_v
vlfeedback
rlaif_v
orca_pairs
nectar_rm
orca_dpo_de
```

## 4. KTO 数据

KTO 数据也不是普通问答格式。它通常是：

```text
一个回答 + 一个好/坏反馈标签
```

示例：

```json
{
  "messages": [
    {
      "role": "user",
      "content": "user question"
    },
    {
      "role": "assistant",
      "content": "model answer"
    }
  ],
  "label": true
}
```

在 `dataset_info.json` 中，这类数据通常有 `kto_tag` 字段映射：

```json
{
  "columns": {
    "messages": "messages",
    "kto_tag": "label"
  }
}
```

代表数据集包括：

```text
kto_en_demo
kto_mix_en
ultrafeedback_kto
```

## 5. 多模态数据

多模态数据可能仍然是“用户问、助手答”的形式，但输入不只有文本，还包含图片、视频或音频。

图片数据示例：

```json
{
  "messages": [
    {
      "role": "user",
      "content": "<image>Describe this image."
    },
    {
      "role": "assistant",
      "content": "This image shows ..."
    }
  ],
  "images": [
    "xxx.jpg"
  ]
}
```

多模态字段包括：

```text
images
videos
audios
```

对应的文本占位符包括：

```text
<image>
<video>
<audio>
```

代表数据集包括：

```text
mllm_demo
mllm_audio_demo
mllm_video_demo
mllm_video_audio_demo
llava_1k_en
llava_1k_zh
llava_150k_en
llava_150k_zh
pokemon_cap
mllm_pt_demo
```

另外，部分偏好排序数据也是多模态的，例如：

```text
rlhf_v
vlfeedback
rlaif_v
```

## 6. 工具调用数据

工具调用数据是对话数据的一种扩展，用于训练模型学会 function calling 或 tool calling。

这类数据通常包含：

```text
messages / conversations
tools
function_call
observation
```

代表数据集包括：

```text
glaive_toolcall_en_demo
glaive_toolcall_zh_demo
glaive_toolcall_en
glaive_toolcall_zh
glaive_toolcall_100k
```

典型字段映射：

```json
{
  "columns": {
    "messages": "conversations",
    "tools": "tools"
  }
}
```

## 总结

`dataset_info.json` 应该理解为：

```text
训练数据注册表
```

而不是：

```text
对话数据集合
```

它包含多种训练数据类型：

```text
普通 SFT 数据       多数像对话/问答
ShareGPT 数据      多轮对话
预训练数据         纯文本，不是对话
偏好数据           chosen/rejected，不是单一输出
KTO 数据           回答 + 反馈标签
多模态数据         文本 + 图像/视频/音频
工具调用数据       对话 + tools/function call
```

如果只看 `identity`、`alpaca_en_demo` 这类样本，确实会觉得它们都是“输入输出对话”。但整个 `dataset_info.json` 不能简单归为全是对话类任务。
