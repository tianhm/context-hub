---
name: package
description: "AG2 / AutoGen Python package for building LLM agents, multi-agent chats, tools, and code-executing workflows"
metadata:
  languages: "python"
  versions: "0.11.2"
  revision: 1
  updated-on: "2026-03-12"
  source: maintainer
  tags: "autogen,ag2,agents,llm,multi-agent,openai,tool-use"
---

# AG2 / AutoGen Python Package Guide

## Golden Rule

Treat `autogen` on PyPI as the AG2 project. The package name in your lockfile may be `autogen` or `ag2`, but the import path in current Python examples is still `autogen`. Install the OpenAI extra when you want OpenAI-backed agents, provide model credentials explicitly, and do not enable code execution without understanding where the code will run.

## Install

Pin the package version your project expects:

```bash
python -m pip install "autogen[openai]==0.11.2"
```

Equivalent commands:

```bash
uv add "autogen[openai]==0.11.2"
poetry add "autogen[openai]==0.11.2"
```

Notes:

- Upstream docs often show `ag2[openai]`. `autogen` and `ag2` are package aliases for the same AG2 project; keep the name already used by the repo you are editing.
- Since AG2 `0.8`, model-client extras are optional. If you skip `[openai]`, OpenAI model calls will fail until you install the matching extra.
- If you need Docker-backed code execution, install Docker separately. AG2 does not bundle the Docker runtime.

## Authentication And Model Setup

For OpenAI-backed agents, set credentials in the environment:

```bash
export OPENAI_API_KEY="your-api-key"
```

Minimal model configuration:

```python
from autogen import AssistantAgent
from autogen.llm_config import LLMConfig

llm_config = LLMConfig(
    api_type="openai",
    model="gpt-4.1-mini",
    api_key="your-api-key",
)

assistant = AssistantAgent(
    name="assistant",
    llm_config=llm_config,
)
```

If credentials already live in the environment, keep secrets out of source files:

```python
import os

from autogen import AssistantAgent
from autogen.llm_config import LLMConfig

llm_config = LLMConfig(
    api_type="openai",
    model="gpt-4.1-mini",
    api_key=os.environ["OPENAI_API_KEY"],
)

assistant = AssistantAgent(name="assistant", llm_config=llm_config)
```

AG2 also supports loading a config list from JSON or an environment variable:

```python
from autogen.llm_config import LLMConfig

llm_config = LLMConfig.from_json(
    path="OAI_CONFIG_LIST",
    file_location=".",
    filter_dict={"model": ["gpt-4.1-mini"]},
)
```

Use that pattern when a project already stores multi-model credentials in `OAI_CONFIG_LIST`.

## Core Usage

### Quick start with a single assistant

`run()` prepares the chat and `process()` executes it:

```python
from autogen import AssistantAgent
from autogen.llm_config import LLMConfig

assistant = AssistantAgent(
    name="assistant",
    llm_config=LLMConfig(
        api_type="openai",
        model="gpt-4.1-mini",
        api_key="your-api-key",
    ),
    system_message="You are a concise Python coding assistant.",
)

chat_result = assistant.run(
    message="Write a Python function that groups strings by first letter."
)
chat_result.process()

print(chat_result.summary)
```

### Add a human proxy or tool-calling peer

Use `UserProxyAgent` when a workflow needs a human checkpoint, tool execution, or code execution policy:

```python
from autogen import AssistantAgent, UserProxyAgent
from autogen.llm_config import LLMConfig

assistant = AssistantAgent(
    name="assistant",
    llm_config=LLMConfig(
        api_type="openai",
        model="gpt-4.1-mini",
        api_key="your-api-key",
    ),
)

user_proxy = UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    code_execution_config=False,
)

result = user_proxy.initiate_chat(
    assistant,
    message="Summarize the advantages of asyncio for IO-bound work.",
)

print(result.summary)
```

### Multi-agent group chat

AG2 is built around agent-to-agent conversations. A minimal planning-and-review flow looks like this:

```python
from autogen import AssistantAgent
from autogen.agentchat.group.patterns import AutoPattern
from autogen.llm_config import LLMConfig

llm_config = LLMConfig(
    api_type="openai",
    model="gpt-4.1-mini",
    api_key="your-api-key",
)

planner = AssistantAgent(name="planner", llm_config=llm_config)
reviewer = AssistantAgent(name="reviewer", llm_config=llm_config)

groupchat = AutoPattern(
    initial_agent=planner,
    agents=[planner, reviewer],
)

result = groupchat.run(
    message="Plan a Python CLI that syncs local markdown files to S3."
)
result.process()
```

Use specialized system prompts and smaller agent sets first. Group chats become noisy quickly when every agent has the same role.

## Code Execution

AG2 can execute code through a configured executor. Keep execution disabled unless you explicitly need it:

```python
from autogen import AssistantAgent, UserProxyAgent
from autogen.coding import LocalCommandLineCodeExecutor
from autogen.llm_config import LLMConfig

executor = LocalCommandLineCodeExecutor(work_dir="coding")

assistant = AssistantAgent(
    name="assistant",
    llm_config=LLMConfig(
        api_type="openai",
        model="gpt-4.1-mini",
        api_key="your-api-key",
    ),
)

user_proxy = UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    code_execution_config={"executor": executor},
)

user_proxy.initiate_chat(
    assistant,
    message="Write and run Python code that prints the first 10 Fibonacci numbers.",
)
```

Notes:

- `LocalCommandLineCodeExecutor` runs code on the host machine. Treat generated code as untrusted.
- Prefer Docker-backed execution when you need stronger isolation and the environment supports it.
- Keep the execution working directory disposable. Do not point it at your repo root.

## Common Pitfalls

- The package slug and the project branding differ. `autogen` on PyPI is AG2, and the docs frequently use `ag2` in install commands.
- The import path does not change just because the install command uses `ag2`; current examples still import from `autogen`.
- Missing extras are a common failure mode. Install `[openai]` or the provider extra that matches your model backend.
- `LLMConfig` needs provider-specific fields. Do not assume a raw `model=` string is enough when the backend also needs `api_type`, `base_url`, or credentials.
- `run()` and `initiate_chat()` return chat results; call `process()` when you want the conversation to execute and finalize under the newer workflow shown in current docs.
- Code execution is opt-in. `code_execution_config=False` is safer than silently allowing local execution.
- Older examples on blogs and issue threads may still use pre-0.9 group-chat or Swarm terminology. Prefer the current AG2 docs for orchestration patterns.

## Version-Sensitive Notes

- The version used here `0.11.2` matches the current PyPI release as of March 12, 2026.
- The docs URL points to the GitHub repo. The canonical maintainer docs for daily use are on `docs.ag2.ai`; use the repo mainly for source, examples, and release history.
- AG2 `0.8` made provider integrations optional extras, which is why current install commands explicitly include `[openai]`.
- AG2 `0.9` merged the older Swarm and Group Chat concepts into a unified Group Chat pattern model. If you see Swarm-only examples, treat them as older guidance unless the target codebase is already built around them.
- The docs root used here is `https://docs.ag2.ai/latest/`, not a version-pinned `0.11.2` snapshot. If an example disagrees with your installed package, check the repo tag or release notes for `v0.11.2` before copying newer APIs.

## Official Sources Used

- PyPI package page: `https://pypi.org/project/autogen/`
- AG2 install guide: `https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/installing-ag2/`
- AG2 quick start: `https://docs.ag2.ai/latest/docs/quick-start/`
- AG2 LLM configuration guide: `https://docs.ag2.ai/latest/docs/user-guide/basic-concepts/llm-configuration/`
- AG2 LLM configuration deep dive: `https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/llm-configuration-deep-dive/`
- AG2 OpenAI model guide: `https://docs.ag2.ai/latest/docs/user-guide/models/openai/`
- AG2 code execution guide: `https://docs.ag2.ai/latest/docs/user-guide/advanced-concepts/code-execution/`
- AG2 repository: `https://github.com/ag2ai/ag2`
