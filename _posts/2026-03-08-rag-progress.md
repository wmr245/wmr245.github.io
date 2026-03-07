---
layout: post
title: "Week2 进展记录：RAG 查询闭环、Gateway 对外链路与基础可观测性"
date: 2026-03-08 10:00:00 +0800
categories: [til]
tags: [C++, Drogon, FastAPI, PostgreSQL, pgvector, Redis, RAG, Docker, LLM]
---

# 背景

本周在完成 RAG 摄取链路之后，继续向“可演示、可扩展的完整闭环”推进。

当前项目仍采用双服务结构：

- **C++ gateway**：负责对外 REST API、文件上传、请求转发、统一入口以及后续工程化能力承接
- **Python AI service**：负责文档摄取、文本分块、向量化、检索、答案生成与任务状态管理
- **PostgreSQL + pgvector**：负责存储文档、任务、分块与向量
- **Redis**：当前已接入环境，预留给后续缓存与限流阶段

本阶段工作的核心目标有四个：

1. 在已完成 ingest 的基础上，实现最小可用查询接口
2. 打通问题 embedding、向量检索、citation 返回与答案生成
3. 将查询、上传与任务状态接口通过 C++ gateway 对外暴露
4. 增加 traceId 与阶段耗时日志，为后续工程化提供可观测基础

本阶段仍然延续相同的工程原则：

> 先把能力闭环跑通，再把工程化手段逐层叠加。

这样做的原因在于，RAG 系统的复杂度往往不是来自单点功能，而是来自多个环节叠加后的联调复杂度。只有先拿到一个可以端到端演示的最小版本，后续缓存、限流、指标、gRPC 替换等工作才有明确依附点。

# 当前阶段的系统职责划分

在这一阶段，进一步明确了 C++ 与 Python 的职责边界。

## C++ gateway 的职责

- 对外提供统一 REST API
- 接收 multipart 文件上传
- 保存上传文件到共享目录
- 转发查询、上传与任务查询请求到内部 AI service
- 作为 traceId 入口与后续限流、缓存、鉴权的承载层

## Python AI service 的职责

- 执行 `/internal/ingest`
- 执行 `/internal/query`
- 创建 `docs` 与 `tasks`
- 在后台触发文档摄取任务
- 从 pgvector 执行向量检索
- 调用 LLM 生成 answer

这种拆分的一个核心好处是：

> Python 负责 AI 能力本身，C++ 负责把 AI 能力服务化、产品化与工程化。

# 查询链路实现

在摄取链路跑通之后，本周首先实现了最小查询接口。

## 目标

先不急于做复杂 Prompt 或多轮能力，而是先把下面这条链路打通：

1. 接收用户问题
2. 对问题执行 embedding
3. 在 `chunks` 表中做 top K 向量检索
4. 按 `docScope` 进行过滤
5. 返回 citation
6. 基于检索结果生成最小答案

## Python 内部查询接口

在 Python AI service 中新增了：

- `POST /internal/query`

请求体支持以下字段：

- `question`
- `docScope`
- `topK`

返回结果包括：

- `question`
- `answer`
- `items`
- `latencyMs`

其中，`items` 中每一项包含：

- `docId`
- `chunkId`
- `chunkIndex`
- `score`
- `text`
- `citation`

## 检索 SQL 的思路

当前检索采用 pgvector 的 cosine distance。核心思路如下：

```sql
SELECT
  c.id,
  c.doc_id,
  c.chunk_index,
  c.text,
  c.page,
  d.title,
  1 - (c.embedding <=> %s::vector) AS score
FROM chunks c
JOIN docs d ON d.id = c.doc_id
WHERE c.embedding IS NOT NULL
  AND d.status = 'ready'
  AND c.doc_id = ANY(%s::bigint[])
ORDER BY c.embedding <=> %s::vector
LIMIT %s
```

如果 `docScope` 为空，则改为在所有 `ready` 文档中检索。

## 实现顺序

这一阶段刻意采用了以下顺序：

1. 先实现只返回 retrieval items 的最小版本
2. 再确认 `docScope`、`topK`、排序结果是否符合预期
3. 最后接入 LLM 生成 `answer`

这样做的目的，是把“检索是否正确”和“生成是否合理”分成两个独立问题处理，避免把 retrieval 与 generation 混在一起排错。

# 查询测试与结果分析

完成 `/internal/query` 后，分别做了单文档检索、多文档检索与全局检索测试。

## 测试请求

```bash
curl -X POST http://localhost:8000/internal/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What does the gateway do?",
    "docScope": [1],
    "topK": 3
  }'
```

以及：

```bash
curl -X POST http://localhost:8000/internal/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What does the gateway do?",
    "topK": 5
  }'
```

## 结果说明

测试结果表明：

- `docScope` 生效，单文档检索时只返回指定文档内容
- `topK` 生效，结果数量与请求参数一致
- citation 能正常返回 `title` 与 `page`
- 英文问题对英文文档的检索得分显著高于中文问题

这里得到一个很有价值的结论：

> embedding 与向量检索链路不仅已经打通，而且排序行为与语言匹配关系符合预期。

也就是说，这一阶段不仅证明“能搜到”，也证明“排序大致合理”。

# 最小答案生成

在确认检索结果正确后，继续接入在线 LLM 生成最小答案。

## 调用方式

当前采用与 embedding 一致的兼容接口风格：

- endpoint：`/chat/completions`
- model：`qwen-plus`

生成函数的职责如下：

1. 将 top K 检索结果拼装为 context
2. 使用一个约束型 prompt，要求模型仅根据上下文回答
3. 如果上下文不足，则明确说明不能确认答案

## 结果

当前 `/internal/query` 已能够返回如下结构：

- `question`
- `answer`
- `items`
- `latencyMs`

这意味着系统已经从“只有 retrieval 的半闭环”，进入到：

> question -> retrieval -> generation -> citations

这样的最小 RAG 闭环。

# C++ Gateway 对外查询接口

在 Python 内部查询可用之后，本周继续把该能力对外暴露到 C++ gateway。

## 新增接口

- `GET /health`
- `POST /rag/query`

其中，`/rag/query` 的职责包括：

1. 校验请求体是否为 JSON
2. 校验 `question` 字段是否存在
3. 透传 `question`、`docScope`、`topK`
4. 调用 Python `/internal/query`
5. 将响应状态码与 JSON 返回给外部调用方

## 测试请求

```bash
curl -X POST http://localhost:8080/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What does the gateway do?",
    "docScope": [1],
    "topK": 3
  }'
```

## 结果

外部 `/rag/query` 已能成功返回：

- answer
- 检索项
- citation
- latency

这说明当前系统已经具备“通过 C++ 暴露统一查询入口”的能力，符合最初的分层设计方向。

# 外部上传与任务状态接口

为了让系统从“能查”进一步变成“能完整演示”，本周继续补上了文件上传与任务查询的外部接口。

## 新增接口

C++ gateway 对外新增：

- `POST /docs/upload`
- `GET /tasks/{taskId}`

Python AI service 内部新增：

- `POST /internal/docs/upload`
- `GET /internal/tasks/{task_id}`

## 上传链路

当前上传链路如下：

1. C++ gateway 接收 multipart 文件
2. 将文件保存到共享上传目录
3. 将 `sourcePath`、`title`、`owner` 转发给 Python AI service
4. Python 创建 `docs` 与 `tasks`
5. Python 通过后台任务执行 ingest
6. 外部通过 `GET /tasks/{taskId}` 查询状态

## 测试请求

```bash
curl -i -X POST http://localhost:8080/docs/upload \
  -F "file=@uploads/sample.md" \
  -F "title=sample_from_gateway.md" \
  -F "owner=demo"
```

随后查询任务状态：

```bash
curl -i http://localhost:8080/tasks/<taskId>
```

## 结果

测试结果表明：

- 文件上传成功
- 返回了新的 `docId` 与 `taskId`
- 任务状态可从 `queued` / `running` 过渡到 `success`
- 新文档 ingest 能通过 gateway 驱动完成

这意味着系统对外已经具备以下主链路：

> 上传 -> 建任务 -> 后台摄取 -> 查询状态 -> 对外问答

这比单纯的内部接口调试前进了一大步。

# traceId 与基础可观测性

在功能闭环已经成立之后，本周开始补入基础可观测性。

## traceId 透传

当前做法如下：

- gateway 读取请求头中的 `x-trace-id`
- 如果请求中没有，则在 gateway 侧生成新的 traceId
- gateway 将 traceId 透传给 Python internal API
- Python 日志在 ingest / query / task 等关键流程中记录 traceId

这样做的价值在于：

> 一次外部请求可以在 gateway 与 Python service 两侧形成同一条日志链路。

这对于后续做缓存命中排查、延迟定位与错误追踪都很有帮助。

## 分阶段耗时日志

在 Python `/internal/query` 中，对以下三个阶段分别打点：

- `embed_ms`
- `retrieve_ms`
- `generate_ms`

同时记录：

- `total_ms`

在 gateway 侧，还通过响应头返回：

- `x-trace-id`
- `x-upstream-ms`

这让当前系统至少具备了初步的延迟拆解能力，而不只是知道一个总延迟数字。

# 延迟现象的分析

本周对查询延迟做了初步观察。

在只做 retrieval 时，查询耗时大约在数百毫秒级。  
在接入 LLM answer generation 之后，端到端查询耗时上升到 2 到 4 秒级。

这是一个符合预期的结果。原因在于：

1. embedding 需要一次外部模型调用
2. generation 又需要一次外部模型调用
3. gateway 内部转发只增加了较小的本地开销

进一步通过对 upload、task 查询与 query 三类请求的对比可知：

- `upload` 的 `x-upstream-ms` 只有数十毫秒
- `task` 查询的 `x-upstream-ms` 也只有数十毫秒
- `query` 总耗时显著更高

据此可以得到当前阶段的判断：

> 系统主要耗时不在 C++ gateway，而在 embedding 与 LLM generation 这两个外部依赖上。

这个结论为后续缓存与性能优化提供了清晰方向。

# 一个小问题：重复样本对检索排序的影响

测试过程中，曾经存在内容重复的文档样本。  
当不传 `docScope` 做全局检索时，重复文档会同时出现在 top K 结果中，看起来像“重复命中”。

这并不是检索 SQL 本身有问题，而是数据集构造导致的自然结果。  
这一现象提醒了一个工程实践点：

> 在做 retrieval 评估或人工观察排序质量时，应尽量清理重复样本，避免对结果判断造成干扰。

# 两个联调问题与修复

本周联调过程中，还遇到了两个典型的工程问题。

## Pydantic 类型顺序问题

在定义 `QueryResponse` 时，如果 `RetrievedItem` 还未声明，会触发：

```text
NameError: name 'RetrievedItem' is not defined
```

修复方式是调整 Pydantic 模型定义顺序，使依赖类型先定义、被引用类型后定义。

## Drogon `setBody` 参数类型问题

在 gateway 中透传 Python JSON 响应时，曾直接使用：

```cpp
outResp->setBody(resp->body());
```

但 `resp->body()` 返回的是 `std::string_view`，而 `setBody()` 需要的是 `std::string` 或字符指针加长度，因此编译失败。

修复方式为：

```cpp
outResp->setBody(std::string(resp->body()));
```

这个问题的本质是 C++ 类型适配问题，而不是 Drogon 逻辑错误。

# 当前项目状态

截至目前，RAG 项目已经完成以下内容：

- 环境与容器编排
- PostgreSQL + pgvector 表结构初始化
- `/internal/ingest` 摄取链路
- 长文本分块验证
- 在线 embedding 接入与向量写入验证
- `/internal/query` 检索链路
- citation 返回
- 在线 LLM 最小 answer generation
- C++ `/rag/query` 对外接口
- C++ `/docs/upload` 对外上传接口
- C++ `/tasks/{taskId}` 对外任务查询接口
- traceId 透传
- query 分阶段耗时日志
- gateway upstream 耗时响应头

因此，当前系统已经不再只是“有 ingest 的骨架”，而是具备了一个最小可演示产品应有的主链路。

# 下一步计划

按照原计划，下一阶段将继续推进 Week3 的工程化内容，优先级如下：

1. Redis query cache
2. retrieval cache
3. 更规范的结构化日志字段
4. gateway 限流
5. 指标记录与简单统计

同时，在架构层面，仍然保留后续替换内部 HTTP 为 gRPC 的演进路径。

# 本周结论

本周工作进一步验证了以下几点：

1. retrieval 与 generation 应分阶段实现和验证
2. C++ gateway 的价值在于统一入口、隔离协议与承接工程化能力
3. 先形成完整外部链路，比过早优化局部实现更重要
4. traceId 与阶段耗时日志是进入工程化阶段的必要基础
5. query 的主要耗时通常来自模型调用，而不是本地 gateway 转发
6. 一个可演示的 RAG 系统不仅要能 ingest，也要能上传、查询、返回 answer 并查询任务状态
