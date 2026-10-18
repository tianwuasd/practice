# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位与同步目标

这是学习 Datawhale Hello-Agents 及其 Python 框架的小型实践项目，主要工作是运行示例、阅读框架源码和做实验。默认用中文沟通。

用户指定的 GitHub 同步目标是 **https://github.com/tianwuasd/practice.git**，本项目应位于该仓库的 **`hello_agent/` 子目录**，与已有的 `medical_image_process/` 并列。本文的路径与命令均相对于 `hello_agent` 项目目录，而不是 Git 仓库根目录。执行版本控制操作前，先确认实际仓库根目录和远程；仅在用户要求时提交或推送，且不得覆盖其他项目。

## 环境与常用命令

以 `environment.yml` 为环境定义：Conda 环境名为 `hello_agent`，Python 为 3.12，框架固定为 `hello-agents==1.0.1`。Conda 管理 Python 环境，pip 从官方 PyPI 安装框架和依赖。

首次创建：

```powershell
conda env create -f environment.yml
conda activate hello_agent
```

已有环境需要按配置更新时：

```powershell
conda env update -n hello_agent -f environment.yml
```

在项目目录中运行示例：

```powershell
conda activate hello_agent
python test.py
```

`test.py` 是直接执行的教学示例，不是单元测试：顶层代码会发送两次模型请求，导入它也会触发这些代码。只有用户要求实际模型验证时才运行；不能把它当作离线测试或默认交给 pytest 收集。

当前没有配置自动化测试套件、构建流程或 lint 工具。修改后可先执行这些不调用模型 API 的检查：

```powershell
python -m pip check
python -c "import ast; from pathlib import Path; ast.parse(Path('test.py').read_text(encoding='utf-8')); print('syntax OK')"
python -c "from hello_agents.tools import CalculatorTool; r = CalculatorTool().run({'input': '2 + 3 * 4'}); assert r.data['result'] == 14; print(r.data['result'])"
```

本机 `.vscode/settings.json` 的解释器配置包含 Windows 绝对路径，已通过 `.gitignore` 排除，不随仓库同步。在其他设备上应自行选择 `hello_agent` 环境，不要假定本机路径通用。

## DeepSeek 配置

`test.py` 先通过 `load_dotenv()` 加载本地 `.env`，随后 `HelloAgentsLLM()` 从进程环境变量读取配置。已有环境变量优先于 dotenv 文件中的同名值。

`.env.example` 提供这些配置项：

- `LLM_MODEL_ID=deepseek-flash`
- `LLM_BASE_URL=https://api.deepseek.com`
- `LLM_API_KEY`：在本地 `.env` 填写。
- `LLM_TIMEOUT=60`

需要首次建立 `.env` 时，仅在文件不存在的情况下复制模板：

```powershell
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

本地 `.env` 保存真实 DeepSeek 凭据；`.gitignore` 已排除 `.env` 和 `.env.*`，并特许提交不含密钥的 `.env.example`。同步到上述 GitHub 仓库时保留这个区别，不要输出 `.env` 内容或把真实密钥复制进说明、源码和模板。

DeepSeek 使用框架的 OpenAI 兼容适配器。适配器名或 `provider` 显示为 `openai` 不代表请求发送给 OpenAI；服务地址由 `LLM_BASE_URL` 决定。

## 运行代码与阅读副本的区别

实际执行 `from hello_agents import ...` 时，加载的是 Conda 环境 `site-packages` 内的包。

`_reference/hello_agents/` 是从已安装的 **1.0.1** 包复制的源码阅读快照，包含包内资源和 `LICENSE`，不包含完整上游仓库的开发配置、测试与文档。放在子目录是为了避免遮蔽已安装的包。

- 修改阅读副本不会改变 `python test.py` 的实际框架行为。
- 不要为使副本生效而擅自修改 `sys.path`、设置 `PYTHONPATH` 或把它搬到项目顶层。
- 如需开发或调试修改后的框架，应先明确切换导入来源；当前快照不是可直接执行 `pip install -e` 的完整上游项目。
- 升级环境里的包不会自动更新快照；更新快照时应同时核对版本并保留许可证。

上游框架仓库是 https://github.com/jjyaoao/HelloAgents；它与用户的 `practice` 同步仓库不是同一仓库。

## 源码阅读主线

从 `test.py` 的模型客户端和 `SimpleAgent` 入手，不必先通读全部扩展模块：

1. `_reference/hello_agents/core/llm.py`：读取模型配置，创建服务适配器，对外提供 `invoke` 等统一调用接口。
2. `_reference/hello_agents/agents/simple_agent.py`：组装系统提示词、历史消息和本次输入；`run()` 消费内部事件，最终返回本次运行的回答。
3. `_reference/hello_agents/core/runtime.py`：`turn_events()` 实现共享的模型与工具循环；收到工具请求后执行工具，把结果追加到消息，再请求模型。
4. `_reference/hello_agents/core/llm_adapters.py`：适配不同服务协议；当前 DeepSeek 配置经 `OpenAIAdapter` 调用 OpenAI SDK 的兼容接口。
5. `_reference/hello_agents/tools/registry.py` 与 `tools/builtin/calculator.py`：分别负责工具注册、查找与执行调度，以及数学表达式的实际计算。

`Agent` 基类负责组装历史管理等组件，`get_history()` 也由基类提供。包顶层的 `__init__.py` 只是将这些类重新导出，方便简短导入。

当前示例虽然创建了 `CalculatorTool`，但 `agent.add_tool(calculator)` 仍被注释，因此第二次提问不构成工具调用验证。1.0.1 已支持 `SimpleAgent.add_tool()`，不要把示例中“需要自行实现 MySimpleAgent”的旧教程注释当作当前版本限制。
