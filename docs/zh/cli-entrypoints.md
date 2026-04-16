# LLaMA Factory CLI 入口说明

本文汇总旧版主线 `launcher` 中几个常用 CLI 命令入口的用途、调用链和典型用法。这里讨论的是未设置 `USE_V1=1` 时的默认入口：

```python
api      -> .api.app.run_api()
chat     -> .chat.chat_model.run_chat()
export   -> .train.tuner.export_model()
train    -> .train.tuner.run_exp()
webchat  -> .webui.interface.run_web_demo()
webui    -> .webui.interface.run_web_ui()
```

入口判断位于 `src/llamafactory/launcher.py`。`launcher` 会先读取命令名，例如 `api`、`chat`、`train`，然后把后面的 YAML、JSON 或命令行参数交给对应函数继续解析。

参数读取逻辑位于 `src/llamafactory/hparams/parser.py`，支持以下形式：

```bash
llamafactory-cli train examples/train_lora/qwen3_lora_sft.yaml
llamafactory-cli train examples/train_lora/qwen3_lora_sft.yaml learning_rate=1e-5
llamafactory-cli chat examples/inference/qwen3_lora_sft.yaml
```

## api

`api -> .api.app.run_api()` 用于启动一个 OpenAI 风格的 HTTP API 服务。

实现位置：

```text
src/llamafactory/api/app.py
```

它会：

1. 创建 `ChatModel()`，加载模型、tokenizer、adapter、推理后端等。
2. 创建 FastAPI 应用。
3. 用 Uvicorn 启动服务，默认监听 `0.0.0.0:8000`。
4. 提供 OpenAI 兼容接口。

主要接口：

```text
GET  /v1/models
POST /v1/chat/completions
POST /v1/score/evaluation
```

`/v1/chat/completions` 支持普通返回和流式返回。流式返回时使用 SSE。

`/v1/score/evaluation` 主要用于 reward model 或打分模型，不是普通生成式聊天模型时使用。

常用环境变量：

```bash
API_HOST=0.0.0.0
API_PORT=8000
API_KEY=your_key
API_MODEL_NAME=my-model-name
FASTAPI_ROOT_PATH=/xxx
```

典型用法：

```bash
llamafactory-cli api examples/inference/qwen3_lora_sft.yaml
```

启动后可以访问：

```text
http://localhost:8000/docs
```

适合场景：把微调后的模型作为服务暴露给其他程序调用，例如前端、业务服务、评测脚本或 OpenAI SDK 兼容调用。

## chat

`chat -> .chat.chat_model.run_chat()` 用于启动命令行交互式聊天。

实现位置：

```text
src/llamafactory/chat/chat_model.py
```

它会：

1. 创建 `ChatModel()`。
2. 根据参数加载推理模型。
3. 在终端里循环读取用户输入。
4. 调用 `stream_chat()` 逐 token 打印模型回复。
5. 保存当前会话历史，直到输入 `clear` 或 `exit`。

内置命令：

```text
clear  清空当前对话历史
exit   退出 CLI 聊天
```

典型用法：

```bash
llamafactory-cli chat examples/inference/qwen3_lora_sft.yaml
```

`ChatModel` 支持多种推理后端：

```text
huggingface
vllm
sglang
ktransformers
```

示例 inference YAML：

```yaml
model_name_or_path: Qwen/Qwen3-4B-Instruct-2507
adapter_name_or_path: saves/qwen3-4b/lora/sft
template: qwen3_nothink
infer_backend: huggingface
trust_remote_code: true
```

适合场景：本地快速验证模型是否能正常推理、LoRA 是否生效、template 是否正确、输出风格是否符合预期。

## export

`export -> .train.tuner.export_model()` 是模型导出入口，主要用于合并 LoRA adapter、保存完整模型、保存 tokenizer/processor，并生成 Ollama Modelfile。

实现位置：

```text
src/llamafactory/train/tuner.py
```

它会：

1. 读取推理/导出参数，即 `get_infer_args()`。
2. 检查必须设置 `export_dir`。
3. 加载 tokenizer / processor。
4. 加载基础模型和 adapter。
5. 如果是 LoRA，在加载模型阶段合并 adapter。
6. 转换导出 dtype。
7. 使用 `save_pretrained()` 保存模型。
8. 保存 tokenizer / processor。
9. 如果是 reward model，复制 value head 权重。
10. 在导出目录生成 `Modelfile`，方便 Ollama 使用。

典型用法：

```bash
llamafactory-cli export examples/merge_lora/qwen3_lora_sft.yaml
```

示例配置：

```yaml
model_name_or_path: Qwen/Qwen3-4B-Instruct-2507
adapter_name_or_path: saves/qwen3-4b/lora/sft
template: qwen3_nothink
trust_remote_code: true

export_dir: saves/qwen3_sft_merged
export_size: 5
export_device: cpu
export_legacy_format: false
```

注意限制：

```text
adapter_name_or_path + export_quantization_bit 不能同时使用
已经量化的模型不能再 merge adapter
```

常规流程是先把 LoRA 合并成完整模型，再考虑后续量化。

适合场景：训练完 LoRA 后，想得到一个可直接部署、上传 Hub、转换格式或给 Ollama 使用的完整模型目录。

## train

`train -> .train.tuner.run_exp()` 是主训练入口。

实现位置：

```text
src/llamafactory/train/tuner.py
```

它会：

1. 读取 YAML / JSON / CLI 参数。
2. 如果传入 `-h` 或 `--help`，展示训练参数帮助。
3. 解析 Ray 参数。
4. 如果 `use_ray: true`，走 Ray 训练。
5. 否则走普通 `_training_function()`。
6. 根据 `stage` 分发到不同训练流程。

训练阶段分发：

```text
stage: pt   -> 预训练 / continued pretraining
stage: sft  -> 监督微调
stage: rm   -> reward model 训练
stage: ppo  -> PPO 强化学习
stage: dpo  -> DPO 偏好优化
stage: kto  -> KTO 偏好优化
```

典型用法：

```bash
llamafactory-cli train examples/train_lora/qwen3_lora_sft.yaml
llamafactory-cli train examples/train_lora/qwen3_lora_dpo.yaml
llamafactory-cli train examples/train_lora/qwen3_lora_reward.yaml
```

多卡时，真正进入 `run_exp()` 之前，`launcher` 可能会先用 `torchrun` 重启自己。触发条件包括：

```text
FORCE_TORCHRUN=1
或检测到多设备，且不是 Ray / KT
```

`train` 是最核心的入口，CLI 训练 YAML 基本都从这里进入。

## webchat

`webchat -> .webui.interface.run_web_demo()` 是轻量版 Web 聊天界面。

实现位置：

```text
src/llamafactory/webui/interface.py
```

它会启动一个 Gradio 页面，但只包含聊天相关 UI，不包含训练、评估、导出这些完整管理功能。

内部调用：

```python
create_web_demo().queue().launch(...)
```

`create_web_demo()` 创建的是 `Engine(pure_chat=True)`，这个模式会在启动时从命令行参数加载模型。

典型用法：

```bash
llamafactory-cli webchat examples/inference/qwen3_lora_sft.yaml
```

常用环境变量：

```bash
GRADIO_SERVER_NAME=0.0.0.0
GRADIO_SHARE=1
GRADIO_IPV6=1
```

适合场景：想给模型开一个浏览器聊天页面，但不需要完整 LLaMA Board 的训练/导出管理能力。

## webui

`webui -> .webui.interface.run_web_ui()` 是完整的 LLaMA Board Web UI。

实现位置：

```text
src/llamafactory/webui/interface.py
```

它会启动完整 Gradio 管理界面，包含这些 Tab：

```text
Train
Evaluate & Predict
Chat
Export
```

和 `webchat` 的区别：

```text
webchat  只做聊天演示，启动时按推理参数加载模型
webui    是完整控制台，在浏览器里选模型、数据集、训练参数、评估、导出
```

典型用法：

```bash
llamafactory-cli webui
```

启动后访问：

```text
http://127.0.0.1:7860
```

Web UI 里的训练不是直接在 `run_web_ui()` 中执行，而是由 `Runner` 组装参数、生成命令并启动子进程管理训练状态。相关逻辑位于：

```text
src/llamafactory/webui/runner.py
```

## 入口选择

```text
想训练模型：          train
想合并/导出模型：     export
想终端快速聊天：      chat
想 HTTP 服务部署：    api
想网页聊天演示：      webchat
想完整浏览器控制台：  webui
```

常见流程：

```bash
llamafactory-cli train examples/train_lora/qwen3_lora_sft.yaml
llamafactory-cli chat examples/inference/qwen3_lora_sft.yaml
llamafactory-cli export examples/merge_lora/qwen3_lora_sft.yaml
llamafactory-cli api examples/inference/qwen3_lora_sft.yaml
```
