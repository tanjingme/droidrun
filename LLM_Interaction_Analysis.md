# Open-DroidRun LLM 交互与推理流程分析
## 1. 概述
DroidRun 是一个利用大语言模型（LLM）驱动 Android 和 iOS 设备进行自动化操作的框架。通过分析源码，我们发现其核心机制在于将 LLM 的文本输出（Tokens）转化为具体的设备控制指令（ADB 命令）。
本报告详细阐述了 **LLM Providers 如何与框架交互** 以及 **大模型推理结果如何转化为手机操作** 的完整流程。
## 2. LLM Providers 交互机制
Open-DroidRun 并不直接为每个 LLM Provider 编写原生 HTTP 请求代码，而是利用了 **LlamaIndex** 这一抽象层。
*   **核心文件**: `droidrun/agent/utils/llm_picker.py`
*   **配置文件**: `droidrun/config_example.yaml` (或用户生成的 `config.yaml`)
**交互流程**:
1.  **配置加载**: 系统启动时读取 YAML 配置文件，获取 `llm_profiles` 部分的配置。每个 Agent（如 Manager, Executor）可以配置不同的 Provider（OpenAI, Anthropic, Gemini, Ollama, DeepSeek）。
2.  **动态加载**: `llm_picker.py` 中的 `load_llm` 函数根据配置的 `provider` 名称，动态导入 `llama_index.llms` 下对应的模块。
    *   例如：配置为 `GoogleGenAI` 时，会自动导入 `llama_index.llms.google_genai`。
    *   配置为 `OpenAI` 时，导入 `llama_index.llms.openai`。
3.  **统一接口**: 所有 LLM 实例最终都遵循 LlamaIndex 的 `LLM` 基类接口。Agent 只需调用 `.chat()` 或 `.complete()` 方法，无需关心底层 API 的差异。
这种设计使得框架具有极高的扩展性，只需安装对应的 `llama-index-llms-*` 包即可支持新的模型。
## 3. 从 Tokens 到手机操作 (Token-to-Action)
将大模型的推理结果（文本 Tokens）转化为手机操作，主要通过 **Manager-Executor** 分层架构或 **CodeAct** 模式实现。这里以最典型的 **Manager-Executor** 模式为例进行解析。
### 3.1 流程总览
1.  **Manager (规划)**: 接收用户指令，输出高层计划 (Plan) 和当前子目标 (Subgoal)。
2.  **Executor (执行)**: 接收子目标，观察屏幕，输出具体的动作指令 (JSON)。
3.  **Parser (解析)**: 将 LLM 的文本响应解析为结构化数据。
4.  **Dispatcher (分发)**: 根据结构化数据调用 Python 函数。
5.  **Driver (驱动)**: Python 函数调用 ADB 工具，通过 USB/TCP 向手机发送信号。
### 3.2 详细步骤分析
#### 第一阶段：Manager 规划
*   **输入**: 用户指令、屏幕截图（可选）、App信 息。
*   **推理**: Manager Agent (由 `droidrun/agent/manager` 实现) 根据 `manager/system.jinja2` prompt 进行思考。
*   **输出**: XML 风格的文本，包含 `<thought>`, `<plan>` 等标签。
*   **传递**: 提取出的当前步骤（Subgoal）被传递给 Executor。
#### 第二阶段：Executor 决策 (关键步骤)
*   **输入**: 当前子目标 (Subgoal)、UI 元素列表、屏幕截图。
*   **Prompt**: 使用 `executor/system.jinja2`。Prompt 明确要求 LLM **“是一个底层的动作执行者”**，并强制要求输出特定格式。
*   **推理结果 (Tokens)**: LLM 输出如下格式的文本：
    ```text
    ### Thought
    用户想打开设置，我看到了 Settings 图标在 index 5。
    ### Action
    {"action": "click", "index": 5}
    ### Description
    点击设置图标
    ```
#### 第三阶段：解析与执行
*   **代码位置**: `droidrun/agent/executor/executor_agent.py` & `prompts.py`
*   **解析**: `parse_executor_response` 函数利用字符串分割 (`split`)提取 `### Action` 后的内容，并解析为 JSON 对象。
    *   提取结果: `{'action': 'click', 'index': 5}`
*   **分发**: `_execute_action` 方法根据 JSON 中的 `action` 字段（如 `click`, `type`, `swipe`）分发逻辑。
    ```python
    if action_type == "click":
        await click(index, tools=self.tools_instance)
    elif action_type == "type":
        await type(text, index, tools=self.tools_instance)
    ```
*   **底层调用**: `click` 函数 (在 `droidrun/agent/utils/tools.py`) 调用 `AdbTools.tap_by_index`。
*   **物理操作**: `AdbTools` 计算坐标，最终通过 `adb shell input tap x y` 真正触碰手机屏幕。
## 4. Prompt 模板分析
系统将 Prompt 模板按功能分类存储在 `droidrun/config/prompts/` 目录下。以下是主要 Prompt 的功能解构：
### 4.1 Executor Prompt (执行器)
**路径**: `config/prompts/executor/system.jinja2`
**功能**: 将自然语言指令转化为原子操作。
*   **角色定义**: "You are a LOW-LEVEL ACTION EXECUTOR... You are a dumb robot." (强调机械执行，不需过度思考)
*   **原子动作定义**: 动态插入 `atomic_actions`，列出可用函数签名，如 `click(index)`, `type(text, index)`.
*   **核心指令**: "Parse the subgoal text literally and execute the matching atomic action."
*   **输出格式强校验**:
    ```text
    ### Action
    Choose only one action... You must provide your decision using a valid JSON format...
    ```
### 4.2 Manager Prompt (管理器)
**路径**: `config/prompts/manager/system.jinja2`
**功能**: 任务拆解、进度跟踪、错误恢复。
*   **思维链 (CoT)**: 要求输出 `<thought>`, `<plan>`, `<add_memory>` 等标签。
*   **上下文管理**: 注入 `<user_request>`, `<device_date>`, `<app_card>` 等上下文信息。
*   **规划能力**: "Please update or copy the existing plan according to the current page..."
### 4.3 CodeAct Prompt (代码执行)
**路径**: `config/prompts/codeact/system.jinja2`
**功能**: 直接生成 Python 代码来操作设备（另一种 Agent 模式）。
*   **输出**: 要求输出 Python 代码块 (` ```python ... ``` `).
*   **工具暴露**: "You can use the following functions: {{ tool_descriptions }}".
*   **执行逻辑**: 框架直接提取代码块并使用 `exec()` 或类似机制在沙箱中运行生成的 Python 代码。
## 总结
DroidRun 通过 **LlamaIndex** 实现多模型兼容，通过 **Manager-Executor 双层架构** 解决复杂任务规划与底层操作的Gap。LLM 的文本输出通过 **Prompt Engineering (强制 JSON 格式)** 和 **正则解析** 被转化为结构化数据，最终映射为 **ADB 命令**，从而实现对手机的控制。
