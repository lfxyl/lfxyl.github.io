---
layout: blog
istop: false
title: "langchain 入门笔记"
category: 知识归纳
background: grey
background-image: https://s2.loli.net/2026/01/17/TW8Yj7qekC9JzB4.png
tags:
- langchain
- ai agent
---

# Ollama 
```bash
# 安装
curl -fsSL https://ollama.com/install.sh | bash
# 检查版本
ollama --version
```

```bash
# 运行模型。如果不存在则自动拉取
ollama run llama3.2 
# 拉取模型。从库中下载模型但不运行 
ollama pull deepseek-r1:14b
# 列出模型。显示本地所有已下载的模型
ollama list
# 删除模型。删除本地已下载的模型
ollama rm llama3
# 显示信息。查看模型的元数据、参数或 Modelfile
ollama show --modelfile llama3
# 查看进程。显示当前正在运行的模型及显存占用
ollama ps
# 启动服务。启动 Ollama 的 API 服务（通常ollama run时后台自动运行）
ollama serve
# 帮助。查看任何命令的帮助信息
ollama help run
# 复制模型。将现有模型复制为新名称（用于测试）
ollama cp llama3 my-model
# 创建模型。根据 Modelfile 创建自定义模型（高级）
ollama create my-bot -f ./Modelfile
# 推送模型。将自定义的模型上传到 ollama.com
ollama push my-username/my-model
```

输入 ollama run 进入聊天界面后，可以使用以`/`开头的快捷指令来控制对话：
- /bye 或 /exit：最重要！ 退出聊天界面，返回命令行。
- /clear：清空当前的上下文记忆（开启一段新的对话）。
- /show info：查看当前模型的详细参数信息。
- /set parameter seed 123：设置随机种子（高级玩法，用于复现结果）。
- /help：在聊天中查看所有可用的快捷键。

以下参数可以用于 run/generate 命令：
```bash
--num-predict <number>    限制输出 token 数
--temperature <float>     控制随机性
--top-k <int>             采样范围
--top-p <float>           核采样
--seed <int>              固定随机性
--format json             输出 JSON
--keepalive <seconds>     会话保持时间
```

API（当 serve 运行时）
REST 端点（默认 http://localhost:11434/api）：

```
(post) /api/generate：文本生成
(post) /api/chat：对话流式接口
(post) /api/pull：远程拉取
(get)  /api/tags：本地模型列表
```

调用示例（curl）：
```bash
curl http://localhost:11434/api/generate \
  -d '{"model":"qwen2.5","prompt":"hello"}'
```

[ollama 模型列表](https://ollama.com/library)

# Langchain
[Langchain](https://python.langchain.com/docs/get_started/introduction.html) 是一个开源的 LLM 应用开发框架，可以快速构建 LLM 应用。

```bash
# 安装langchain
pip install langchain
# 安装openai集成
pip install langchain-openai
```

```python
from langchain_openai import ChatOpenAI

# 配置本地模型的 API 地址和密钥
llm = ChatOpenAI(
    openai_api_base=f'http://localhost:11434/api/v3',
    openai_api_key=f'ACCESSCODE ...',  # app_key
    model_name="DeepSeek-R1",   
)

# 调用模型并获取结果
result = llm.invoke("你好，怎么称呼？")
print(result)
```


