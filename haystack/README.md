# Haystack

官网：<https://haystack.deepset.ai>  
GitHub：<https://github.com/deepset-ai/haystack>  
文档：<https://docs.haystack.deepset.ai>

## 是什么

Haystack 是一个开源的 LLM 应用框架，由德国公司 deepset 开发。核心能力是构建 RAG（检索增强生成）流水线，将大模型与外部数据源（文档、数据库、API）连接起来。

## 核心概念

### Pipeline

Haystack 的核心抽象是 `Pipeline`——一个有向图，节点是组件（Component），边是数据流。

```python
from haystack import Pipeline
from haystack.components.retrievers import InMemoryEmbeddingRetriever
from haystack.components.readers import PromptBuilder

pipe = Pipeline()
pipe.add_component("retriever", InMemoryEmbeddingRetriever(...))
pipe.add_component("reader", PromptBuilder(...))
pipe.connect("retriever", "reader")
```

### Components

每个组件是一个独立的处理单元。官方提供了大量预置组件：

- **检索**：ElasticsearchRetriever、ChromaEmbeddingRetriever、InMemoryEmbeddingRetriever
- **生成**：OpenAIGenerator、HuggingFaceAPIGenerator
- **转换**：DocumentSplitter、TextCleaner、MetadataFilter
- **存储**：DocumentStore（InMemory、Elasticsearch、Chroma、Weaviate）

### Document Store

文档存储层，支持多种后端。每个 Document 包含 `content`（文本）和 `meta`（元数据字典）。

## 与当前项目的关联

| 场景 | 说明 |
|------|------|
| Agent 记忆管理 | DocumentStore 可用于存储对话历史、用户画像 |
| 工具调用增强 | RAG pipeline 可赋予 Agent 检索外部文档的能力 |
| 模型无关 | Haystack 支持 OpenAI、HuggingFace、Ollama 等多种后端的 generator |

## 快速开始

```bash
pip install haystack-ai
```

```python
from haystack import Pipeline
from haystack.components.generators import OpenAIGenerator

pipe = Pipeline()
pipe.add_component("llm", OpenAIGenerator(model="gpt-4"))
result = pipe.run({"llm": {"prompt": "Hello!"}})
print(result["llm"]["replies"][0])
```

## 注意

- `haystack-ai` 是 Haystack 2.x 的包名（最新版），旧版 `haystack` 是 1.x
- 组件配置通过构造参数传入，非运行时参数通过 `run()` 的 `kwargs` 传入
