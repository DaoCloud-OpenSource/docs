# vLLM 开发入门

作者：noooop

大家好！如果你对 vLLM 项目感兴趣，这里有一份来自社区成员的贡献指南，希望能帮你顺利上手。

## 如何为 vLLM 做出贡献

首先感谢你有兴趣为 vLLM 做出贡献！我们的社区向所有人开放，并欢迎各种形式的贡献，无论大小。你可以通过以下几种方式为项目做出贡献：

- 发现并报告问题或错误。
- 请求或添加对新模型的支持。
- 建议或实现新功能。
- 改进文档或贡献操作指南。

我们也相信社区支持的力量；因此，回答问题、提供 PR 审查以及帮助他人也是备受重视且有益的贡献。

最后，支持我们最有影响力的方式之一是提高对 vLLM 的认识。在你的博客文章中谈论它，并强调它如何推动你的出色项目。如果你正在使用 vLLM，请在社交媒体上表达你的支持，或者通过为我们的仓库加星来表达你的赞赏！

## Job Board 任务板

如果不确定从哪里开始？可以去任务板查看待处理的任务：

- [入门问题](https://github.com/vllm-project/vllm/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22)
- [新模型支持](https://github.com/vllm-project/vllm/issues?q=is%3Aopen+is%3Aissue+label%3Anew-model)

## 🚀 第一步：准备好开发环境

### 1. 把代码搞到本地

```bash
git clone https://github.com/vllm-project/vllm.git
cd vllm
```

### 2. 配置 Python 环境

推荐使用 [uv](https://docs.astral.sh/uv/getting-started/installation/) 管理环境，更轻量快速。安装 uv 后，一键创建环境：

```bash
uv venv --python 3.12 --seed
source .venv/bin/activate
```

> **注意**：建议使用 Python 3.12，vLLM 的主要测试和兼容性基于此版本，可减少本地与测试环境不一致的问题。vLLM 兼容 Python 3.10 至 3.13 版本，但 CI 中的测试（mypy 除外）使用 Python 3.12 运行。

### 3. 安装 vLLM（两种方式）

- 如果**只改 Python 代码**，可开启预编译加速安装：

```bash
VLLM_USE_PRECOMPILED=1 uv pip install -e .
```

- 如果还需要**修改底层 C++/CUDA 内核代码**，使用标准安装方式：

```bash
uv pip install -e .
```

有关从源代码安装以及为其他硬件安装的更多详细信息，请查看针对你硬件的[安装说明](https://docs.vllm.ai/en/latest/getting_started/installation.html)，并转到"从源代码构建 wheel"部分。

## 🔧 开发与调试小贴士

### Debug 调试

如果你熟悉使用 VS Code，可以直接复制下面提供的 `launch.json` 配置，一键启动带调试的 vLLM 服务：

```json
{
   "version": "0.2.0",
   "configurations": [
      {
         "name": "vllm server single",
         "type": "debugpy",
         "request": "launch",
         "module": "vllm.entrypoints.cli.main",
         "env": {
            "VLLM_LOGGING_LEVEL": "DEBUG",
            // "VLLM_USE_MODELSCOPE": "True",
            // "MODELSCOPE_DOWNLOAD_PARALLELS": "10",
         },
         "args": [
            "serve",
            "Qwen/Qwen3-0.6B",
            "--reasoning-parser",
            "qwen3",
            "--gpu-memory-utilization",
            "0.8",
            "--port",
            "8000",
            "--enforce-eager",
            "--max-model-len",
            "5120",
            "-tp",
            "1",
         ],
      },
   ]
}
```

### Profiling 性能分析

vLLM 提供了 `--profiler-config` 参数来开启 profiling：

```bash
vllm serve Qwen/Qwen3-0.6B --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile"}'
```

设置该参数后，vLLM 会启动两个 HTTP endpoint：`/start_profile` 和 `/stop_profile`，你可以通过调用这两个 endpoint 来启动和停止 profiling：

```bash
# 首先调用 /start_profile 接口开始记录
curl -X POST http://localhost:8000/start_profile

# 调用模型生成接口
curl -X POST http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "Qwen/Qwen3-0.6B",
        "messages": [
            {
                "role": "user",
                "content": "9.11 and 9.8, which is greater?"
            }
        ]
    }'

# 调用 /stop_profile 接口停止记录
curl -X POST http://localhost:8000/stop_profile
```

调用 `/stop_profile` 后，`./vllm_profile` 目录下会生成如下文件：

```
1767687660802231168-rank-0.1767687963353114465.pt.trace.json.gz
1767687660802231168-rank-0.1767687964014426797.pt.trace.json.gz
1767687660802231168-rank-0.1767687964637556322.pt.trace.json.gz
1767687660802231168-rank-0.1767687966207743877.pt.trace.json.gz
```

生成的 `.pt.trace.json.gz` 文件可拖到 [https://ui.perfetto.dev/](https://ui.perfetto.dev/) 进行可视化，查看函数调用和耗时详情。

![Profiling 可视化示例](https://github.com/user-attachments/assets/03043b5f-c15c-41be-811a-629f6bf6a1eb)

可以看到非常详细的函数调用过程、耗时等，perfetto.dev 网站支持对每一个地方进行放大，你可以非常细节地观察 vLLM 是如何工作的。

### Linting 代码规范检查

vLLM 使用 [pre-commit](https://pre-commit.com/#usage) 来检查和格式化代码库。设置 pre-commit 非常简单：

```bash
uv pip install pre-commit
pre-commit install
```

现在，每次提交代码时，vLLM 的 pre-commit 钩子都会自动运行。

你也可以手动运行 pre-commit 钩子：

```bash
pre-commit run     # 对暂存文件运行
pre-commit run -a  # 对所有文件运行（--all-files 的缩写）
```

部分 pre-commit 钩子仅在 CI 中运行。如果需要，你可以在本地运行它们：

```bash
pre-commit run --hook-stage manual markdownlint
pre-commit run --hook-stage manual mypy-3.10
```

## 📖 写文档和跑测试

### 文档

vLLM 文档基于 [MkDocs](https://www.mkdocs.org/) 编写，源文件为 Markdown，通过单个 YAML 配置文件 `mkdocs.yaml` 进行配置。

安装依赖：

```bash
uv pip install -r requirements/docs.txt
```

> **提示**：请确保你的 Python 版本与插件兼容（例如，mkdocs-awesome-nav 需要 Python 3.10+）。

MkDocs 自带一个内置的开发服务器，可以在你处理文档时实时预览。在仓库根目录下运行：

```bash
mkdocs serve                           # 包含 API 参考（约 10 分钟）
API_AUTONAV_EXCLUDE=vllm mkdocs serve  # 关闭 API 参考（约 15 秒）
```

当你在日志中看到 `Serving on http://127.0.0.1:8000/` 时，实时预览就准备好了！

### 测试

vLLM 使用 pytest 测试代码库。

```bash
# 安装 CI 中使用的测试依赖项（仅限 CUDA）
uv pip install -r requirements/common.txt -r requirements/dev.txt --torch-backend=auto

# 安装一些通用的测试依赖项（与硬件无关）
uv pip install pytest pytest-asyncio

# 运行所有测试
pytest tests/

# 对单个测试文件运行测试并输出详细信息
pytest -s -v tests/test_logger.py
```

> **提示**：如果缺少 `Python.h`，请使用 `sudo apt install python3-dev` 安装 python3-dev。

> **警告**：目前，并非所有单元测试在 CPU 平台上都能通过。如果你没有 GPU 平台在本地运行单元测试，请暂时依赖持续集成系统来运行测试。

## 📬 正式贡献流程：提交 Issue 与 PR

### 提交 Issue（问题或需求）

- 遇到 Bug 或有新想法？请先到 [GitHub Issues](https://github.com/vllm-project/vllm/issues) 搜索是否已有类似内容。
- 若没有，新建一个 Issue，尽量详细描述（如复现步骤、环境信息）。
- **重要**：如发现安全问题，请按[安全指南](https://github.com/vllm-project/vllm/security/policy)私下报告，勿公开在 Issue 中。

### 提交 Pull Request（PR）

代码完成后即可提交 PR。为保障流程顺畅，请遵守以下约定：

#### DCO 和 Signed-off-by

向本项目贡献更改时，你必须同意 [DCO](https://github.com/vllm-project/vllm/blob/main/DCO)。提交的 commit 必须包含 `Signed-off-by:` 标头，以证明你同意 DCO 的条款。

使用 `-s` 选项执行 `git commit` 将自动添加此标头：

```bash
git commit -s
```

你也可以通过 IDE 启用自动签名：

- **PyCharm**：在提交窗口中点击 "Show Commit Options"，启用 "Sign-off commit"。
- **VSCode**：打开设置编辑器并启用 `Git: Always Sign Off (git.alwaysSignOff)` 字段。

#### PR 标题和分类

PR 标题应添加适当的前缀以指示更改类型：

| 前缀 | 说明 |
|------|------|
| `[Bugfix]` | 修复 Bug |
| `[CI/Build]` | 构建或持续集成改进 |
| `[Doc]` | 文档修复和改进 |
| `[Model] 模型名` | 添加新模型或改进现有模型 |
| `[Frontend]` | vLLM 前端的更改（如 OpenAI API 服务器） |
| `[Kernel]` | 影响 CUDA 内核或其他计算内核的更改 |
| `[Core]` | vLLM 核心逻辑的更改（如 LLMEngine、Scheduler） |
| `[Hardware][Vendor]` | 硬件特定的更改（如 `[Hardware][AMD]`） |
| `[Misc]` | 不符合上述类别的 PR（请谨慎使用） |

> **注意**：如果 PR 涉及多个类别，请包含所有相关前缀。

#### 代码质量要求

PR 需要满足以下代码质量标准：

- 遵循 [Google Python 风格指南](https://google.github.io/styleguide/pyguide.html) 和 [Google C++ 风格指南](https://google.github.io/styleguide/cppguide.html)。
- 通过所有 linter 检查。
- 代码需要充分文档化，以确保未来的贡献者能够轻松理解。
- 包含足够的测试，以确保项目的正确性和健壮性。
- 如果 PR 修改了 vLLM 面向用户的行为，请向 `docs/` 添加文档。

#### Kernels 添加或更改内核

当积极开发或修改内核时，强烈建议使用[增量编译工作流程](https://docs.vllm.ai/en/latest/development/incremental_compilation_workflow.html)以获得更快的构建时间。每个自定义内核都需要一个模式以及一个或多个实现，并在 PyTorch 中注册。

- 确保自定义操作符遵循 PyTorch 指南注册。
- 返回张量的自定义操作需要元函数，应在 Python 中实现并注册。
- 使用 `torch.library.opcheck()` 测试任何已注册操作符的函数注册和元函数。

#### RFC 大型更改注意事项

请尽量保持更改简洁。对于重大的架构更改（> 500 LOC，不包括内核/数据/配置/测试），我们期望有一个 GitHub Issue（RFC）来讨论技术设计和理由。否则，PR 可能会被标记为 `rfc-required` 而暂缓审查。

## 🔍 审查流程（Review）

1. PR 提交后，将被分配给一名审查者。你可以在 [vLLM 贡献者列表](https://docs.vllm.ai/en/latest/governance/committers/) 查看各领域的 Reviewer。
2. 审查者将每 2–3 天提供一次状态更新。如果 PR 在 7 天内未被审查，请随时联系审查者，或到 Slack 的 `#pr-reviews` 频道提醒。
3. 审查后，如果需要更改，审查者会在 PR 上打上 `action-required` 标签。修改完成后请通知审查者重新审查。
4. 请在合理的时间内回应所有评论。如有疑问或不同意见，随时提问讨论。
5. 注意，由于计算资源有限，并非所有 CI 检查都会立即执行。当 PR 准备好合并时，审查者会添加 `ready` 标签，触发完整 CI 检查。

## 🎉 开始你的 vLLM 贡献之旅吧！

关键步骤回顾：

**领任务 → 搭环境 → 写代码 → 跑测试/检查 → 提 PR → 根据反馈修改 → 合并！**

不必担心起点高低，社区伙伴都很友好。从一个小的修复或文档改进开始，就是很好的第一步。期待看到你的第一个 vLLM PR！

## 参考

- <https://docs.vllm.ai/en/latest/contributing/>
- <https://docs.vllm.ai/en/latest/governance/collaboration/>
- <https://docs.vllm.ai/en/latest/governance/process/>
- <https://docs.vllm.ai/en/latest/governance/committers/>
