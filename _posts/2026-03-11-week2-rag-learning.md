---
layout: post
title: "Week2 进展记录"
date: 2026-03-11 10:00:00 +0800
categories: [til]
tags: [C++, Drogon, FastAPI, PostgreSQL, pgvector, Redis, RAG, Docker, LLM]
---

## 今日目标

围绕当前 RAG Knowledge Base 项目，继续推进 Week3 工程化第一步，重点完成或验证以下事项：

1. 验证新上传文档 `docId=4` 是否真的可检索
2. 修复 gateway 返回重复 `content-type` 的问题
3. 设计并接入 Redis query cache
4. 继续推进 cache 失效策略，验证 ingest 成功后能否让旧 query cache 失效

---

## 当前项目背景（今日讨论上下文）

项目当前架构：

- 外部：C++ Drogon Gateway
- 内部：Python FastAPI AI Service
- 数据层：PostgreSQL + pgvector
- Redis：用于 query cache，后续可扩展到限流/缓存
- 当前内部通信：HTTP（暂不切 gRPC）

职责划分：

- Python 负责 AI 能力：ingest / embedding / retrieval / generation
- C++ 负责外部 REST API、文件上传、traceId、后续缓存/限流/鉴权

---

## 今日已确认完成的工作

### 1. 新上传文档 `docId=4` 已确认可检索

通过外部 `/rag/query` 配合 `docScope:[4]` 验证，结果返回：

- `items` 非空
- `items[0].docId = 4`
- `citation.title = sample_from_gateway.md`

这说明：

- `docId=4` 已成功进入检索库
- `docScope:[4]` 生效
- 之前“全局 query 更偏向命中 docId=1”大概率只是相关性排序，不是 bug

---

### 2. Redis query cache 基本链路已经跑通

当前已经完成并验证：

- FastAPI 启动时可连上 Redis
- `/internal/query` 支持：
  - 读缓存
  - miss 后走 embedding / retrieval / generation
  - 写回 Redis（SETEX）
- 对同一个 query 连续请求时：
  - 第一次会 `cache_set`
  - 第二次会 `cache_hit`

已验证的现象：

- 第一次 `What is RAG` 返回正常，耗时约 2.2s
- 第二次相同 query 返回 `latencyMs=1`
- `ai-service` 日志明确出现：
  - `internal_query cache_set ...`
  - `internal_query cache_hit ...`

这说明 query cache 的基本闭环是成立的。

---

### 3. cache key 已升级到带知识库版本号

目前已经把 query cache key 改成了版本化形式，例如：

- `rag:query:v1:kbv0:<sha256>`

并且已经确认：

- Redis 初始 `rag:kb:version` 为 `nil`
- 首次 query 后会初始化为 `0`
- Redis 中扫描到的 key 为：
  - `rag:query:v1:kbv0:...`

这说明：

- 默认版本从 `1` 改成 `0` 之后，初始化边界已修正
- “nil -> 0” 是 query 初始化行为，不是 ingest bump

---

### 4. gateway 修复方案已经产出

已经给出完整的 `cpp-gateway/src/main.cc` 修复版，目标包括：

1. 修复重复 `content-type`
2. 透传 Python 返回的：
   - `x-cache`
   - `x-kb-version`
3. 统一 `/rag/query`、`/docs/upload`、`/tasks/{id}` 的代理响应逻辑

当前状态说明：

- 修复代码已给出
- 但是否已经在本地成功替换、重建并验证，还需要下一轮继续确认

---

## 今日定位出的关键问题

### 问题 1：gateway 之前确实存在重复 `content-type`

外部 `/rag/query` 响应头曾出现：

- `content-type: text/html; charset=utf-8`
- `content-type: application/json`

根因已定位：

- `newHttpResponse()` 默认带 `text/html`
- 又手工 `addHeader("Content-Type", "application/json")`
- 导致重复头

修复原则已明确：

- 改为 `setContentTypeCode(drogon::CT_APPLICATION_JSON)`
- 不再手工追加 `Content-Type`

---

### 问题 2：gateway 之前没有透传 `x-cache` / `x-kb-version`

外部 `/rag/query` 看不到 hit/miss，也看不到当前知识库版本。

根因已定位：

- gateway 只回传了：
  - `x-trace-id`
  - `x-upstream-ms`
- 没有把 Python upstream 的缓存观测头透传出来

修复方案已明确：

- 在 gateway 成功代理响应中透传：
  - `x-cache`
  - `x-kb-version`

---

### 问题 3：ingest 成功后版本号没有真正增加，导致旧 cache 没失效

这是今天最关键的新发现。

现象：

- 上传新文件后，`/tasks/{taskId}` 轮询到 `success`
- 但 Redis 中 `rag:kb:version` 仍然是 `0`
- 同一个 query 仍然 `hit` 旧的 `kbv0` cache

进一步查看日志后发现：

- `run_ingest_job success ...`
- `kb_version bumped ... kb_version=<coroutine object Redis.execute_command ...>`

这说明：

### 真正根因

`bump_kb_version_sync()` 并没有真正执行同步自增，而是把一个 **协程对象** 当成了返回值。

也就是说：

- bump 调用了异步 Redis client（或异步方法）
- 但当前 `run_ingest_job()` 是同步函数
- 结果 `incr(...)` 没有被真正执行
- Redis 中的 `rag:kb:version` 仍停留在 `0`
- 所以 ingest 后 query 还是命中旧版本 cache

### 当前结论

这不是 query 侧 key 设计的问题，也不是 bump 放置顺序的问题，而是：

> `bump_kb_version_sync()` 的实现本身有问题：同步函数里用了异步 Redis API，实际没有完成 `INCR`

---

## 今日看到的关键代码路径

### `/internal/docs/upload`

已确认上传链路会走后台任务：

- `create_doc_and_task(...)`
- `background_tasks.add_task(run_ingest_job, ...)`

说明：

- 外部 `/docs/upload`
- -> Python `/internal/docs/upload`
- -> 后台调用 `run_ingest_job(...)`

因此：

- 把 bump 放在 `run_ingest_job()` 成功尾部，这个位置本身是合理的
- 现在版本不变，主要不是“路径没走到”
- 而是 bump 函数没真正执行成功

---

## 今日最终确认的事实清单

### 已确认正确

- `docId=4` 可检索
- `docScope:[4]` 生效
- Redis query cache 的读写闭环成立
- query cache key 已切成 `kbv0` 版本化形式
- `nil -> 0` 是正常初始化
- 上传链路确实会调用 `run_ingest_job(...)`
- `run_ingest_job()` 日志显示执行到了 success
- bump 调用确实被执行到了，但返回了 coroutine object

### 尚未最终闭环

- `bump_kb_version_sync()` 需要修成真正同步可执行的 `INCR`
- ingest 成功后需要验证：
  - `rag:kb:version` 从 `0 -> 1`
  - 新版本下第一次相同 query 为 `miss`
  - 第二次相同 query 为 `hit`
- gateway 修复版 `main.cc` 是否已替换并验证：
  - 单一 `content-type`
  - 外部可见 `x-cache`
  - 外部可见 `x-kb-version`

---

## 下一步最小改动建议

### 1. 优先修 `bump_kb_version_sync()`

目标：

- 不要在同步函数里直接调用异步 Redis API
- 要么换成同步 Redis client
- 要么在同步上下文里使用明确可执行的同步 `INCR`

优先推荐：

- 在 bump helper 里显式使用 `redis.Redis.from_url(...)`
- 然后同步执行 `incr(KB_VERSION_KEY)`

修完后需要重新验证：

- 上传新文件
- task success
- `rag:kb:version == 1`

---

### 2. 然后验证版本化 cache 失效

预期正确行为：

1. 旧版本 `kbv0`
   - query 第一次 `miss`
   - query 第二次 `hit`

2. 上传新文件后 bump 到 `1`
   - 同样 query 第一次应 `miss`
   - `x-kb-version: 1`
   - 再查第二次才 `hit`

---

### 3. 最后收口 gateway 观测

如果 gateway 修复版已落地，外部 `/rag/query` 应该直接能看到：

- `content-type: application/json`（仅一条）
- `x-cache: hit/miss`
- `x-kb-version: 0/1/...`

这样后续排查就不再依赖 ai-service 日志。

---

## 今日学习收获

1. **RAG 工程化里，cache 本身跑通不难，难的是失效策略**
   - 只做 query cache 很快能通
   - 但 ingest 后如何让旧 cache 失效，才是真正的工程问题

2. **版本化 cache key 是一种非常实用的最小失效策略**
   - 不需要扫描和删除所有旧 key
   - 只要 bump 版本即可切换命名空间

3. **同步/异步边界是 Python 服务中很容易踩坑的地方**
   - 当前问题的根因就是同步 ingest 路径里错误使用了异步 Redis API
   - 日志里出现 `<coroutine object ...>` 是非常强的信号

4. **代理层问题要和业务层问题分开排查**
   - gateway 负责透传和协议层包装
   - Python 负责 cache 语义和版本控制
   - 先分层验证，再联调，效率更高

---

## 当前最核心待办（按优先级）

1. 修复 `bump_kb_version_sync()`，确保 ingest success 后版本真正 `INCR`
2. 重新跑版本失效测试，确认 `0 -> 1`
3. 验证 gateway 修复版是否外部能看到：
   - 单一 `content-type`
   - `x-cache`
   - `x-kb-version`

