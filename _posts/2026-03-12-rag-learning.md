---
layout: post
title: "Week2 进展记录"
date: 2026-03-12 10:00:00 +0800
categories: [til]
tags: [C++, Drogon, FastAPI, PostgreSQL, pgvector, Redis, RAG, Docker, LLM]
---


# 2026-03-11 RAG 项目学习与排障日志

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

- Python 负责 AI 能力：ingest / embedding / retrieval / generation / cache
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

- 第一次 query 返回正常，走完整 embedding / retrieval / generation
- 第二次相同 query 命中缓存，延迟显著下降
- `ai-service` 日志明确出现：
  - `internal_query cache_set ...`
  - `internal_query cache_hit ...`

这说明 query cache 的基本闭环是成立的。

---

### 3. cache key 已升级到带知识库版本号

目前 query cache key 已改成版本化形式，例如：

- `rag:query:v1:kbv0:<sha256>`

并且已经确认：

- Redis 初始 `rag:kb:version` 为 `nil`
- 首次 query 后会初始化为 `0`
- Redis 中实际出现的是：
  - `rag:query:v1:kbv0:...`

这说明：

- 默认版本从 `1` 改成 `0` 后，初始化边界已修正
- “nil -> 0” 是 query 初始化行为，不是 ingest bump

---

### 4. gateway 修复方案已经落地并完成验证

`cpp-gateway/src/main.cc` 修复版已完成落地，目标包括：

1. 修复重复 `content-type`
2. 透传 Python 返回的：
   - `x-cache`
   - `x-kb-version`
3. 统一 `/rag/query`、`/docs/upload`、`/tasks/{id}` 的代理响应逻辑

最终外部 `/rag/query` 已验证可见：

- `content-type: application/json; charset=utf-8`
- `x-kb-version: 1`
- `x-cache: miss`
- `x-upstream-ms: ...`
- `x-trace-id: ...`

并且响应头中只出现了一条 `content-type`。

这说明：

- gateway 重复 `content-type` 问题已修复
- upstream 的缓存观测头已成功透传到外部
- 后续排查 query cache 状态时，不再完全依赖 ai-service 日志

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

修复原则已明确并已验证生效：

- 改为 `setContentTypeCode(drogon::CT_APPLICATION_JSON)`
- 不再手工追加 `Content-Type`

---

### 问题 2：gateway 之前没有透传 `x-cache` / `x-kb-version`

外部 `/rag/query` 一开始看不到 hit/miss，也看不到当前知识库版本。

根因已定位：

- gateway 只回传了：
  - `x-trace-id`
  - `x-upstream-ms`
- 没有把 Python upstream 的缓存观测头透传出来

修复方案已明确并已验证：

- 在 gateway 成功代理响应中透传：
  - `x-cache`
  - `x-kb-version`

最终外部已实际看到：

- `x-cache: miss`
- `x-kb-version: 1`

说明该问题已闭环。

---

### 问题 3：ingest 成功后版本号没有真正增加，导致旧 cache 没失效

这是今天最关键的新发现，也是最终成功修复的问题。

#### 初始现象

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

### 最终修复方式

采用最小改动策略修复：

- 保留原有异步 Redis client，继续给 `/internal/query` 使用
- 额外增加同步 Redis client
- 将 `bump_kb_version_sync()` 改为通过同步 client 执行真正的 `INCR`

修复后再次 ingest，日志变为：

- `run_ingest_job success ...`
- `kb_version bumped ... kb_version=1`

同时 Redis 中确认：

- `GET rag:kb:version`
- 返回 `"1"`

这说明：

> `bump_kb_version_sync()` 已经从“错误调用异步 API 的伪 bump”，修成了真正生效的同步版本递增

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
- 当前版本不变的问题，根因不在“路径没走到”
- 而在 `bump_kb_version_sync()` 实现错误

### `run_ingest_job(...)` 成功路径

当前成功路径逻辑已经确认有效：

1. ingest 正常完成
2. `update_task_status(..., "success", 100, "")`
3. `update_doc_status(..., "ready")`
4. 记录 `run_ingest_job success ...`
5. 调用 `bump_kb_version_sync(...)`
6. Redis 中 `rag:kb:version` 真正递增

这说明 ingest 成功后 bump 的调用时机是正确的。

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
- 之前 bump 调用确实被执行到了，但返回了 coroutine object
- `bump_kb_version_sync()` 已修复为真正同步执行的 `INCR`
- ingest success 后 `rag:kb:version` 已可真实从 `0 -> 1`
- ingest 后相同 query 已切换到 `kbv1`
- gateway 外部已可看到：
  - 单一 `content-type`
  - `x-cache`
  - `x-kb-version`

### 今日最终闭环结果

版本化 cache 失效策略已完成闭环验证：

1. 旧版本 `kbv0`
   - 第一次 query：`miss`
   - 第二次相同 query：`hit`

2. 上传新文件并 ingest success 后
   - `rag:kb:version` 从 `0 -> 1`

3. 相同 query 在新版本下
   - 第一次请求：`miss`
   - 响应头可见 `x-kb-version: 1`
   - ai-service 日志出现 `cache_set ... kb_version=1`

4. 再次请求相同 query
   - 命中 `kbv1` cache
   - 响应头应为 `x-cache: hit`
   - 说明 query cache 已按知识库版本切换命名空间

这说明：

> “ingest 成功后旧 query cache 自动失效” 这一目标，已经通过“版本化 key + 成功后 bump kb version”的方式达成。

---

## 今日最终结论

今天最重要的成果不是“把 Redis 接上了”，而是把 **RAG query cache 的工程化闭环** 真正做出来了：

- cache 能正常命中
- cache key 带版本号
- ingest 成功后版本可递增
- 新版本查询不会继续误命中旧 cache
- gateway 也能把缓存状态和知识库版本直接透传到外部

这意味着项目已经从“单纯能跑通主链路”，推进到了“具备最基本工程可观测性和 cache 失效能力”的阶段。

---

## 下一步最小改动建议

### 1. 先不再大改 query cache 机制

当前最小可用方案已经成立：

- `rag:query:v1:kbvN:<sha256>`
- ingest success 后 bump `rag:kb:version`

这个策略已经足够支撑当前阶段继续开发，不需要立刻引入更复杂的 cache 清理逻辑。

---

### 2. 可以补一层过期 key 的后台清理策略（非阻塞）

虽然版本切换后旧 key 不再命中，但历史 `kbv0`、`kbv1` key 会在 TTL 到期前继续留在 Redis 中。

当前因为已经使用 `SETEX`，所以问题不大；后续可根据需要考虑：

- 保持短 TTL
- 或定期清理旧版本命名空间

但这属于优化项，不是当前阻塞项。

---

### 3. 下一步可以继续推进 gateway 工程能力

在 query cache 已完成最小闭环后，后续更值得推进的是 gateway 侧的工程化能力，例如：

- 限流
- 统一错误码
- 统一 trace / observability
- 上传鉴权
- 更清晰的代理封装

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
   - 最小修复方式通常不是“硬包一层”，而是明确区分 sync client 和 async client

4. **代理层问题要和业务层问题分开排查**
   - gateway 负责透传和协议层包装
   - Python 负责 cache 语义和版本控制
   - 先分层验证，再联调，效率更高

5. **观测头透传非常重要**
   - 外部能直接看到 `x-cache`、`x-kb-version`
   - 会极大降低后续排查成本
   - 这是工程化收益很高、改动却不大的优化点

---

## 当前最核心待办（按优先级）

1. 将今天这套版本化 cache 失效方案固化到项目文档中
2. 补充一组可复用的回归测试命令，覆盖：
   - `kbv0` 第一次 miss / 第二次 hit
   - ingest 后 `0 -> 1`
   - `kbv1` 第一次 miss / 第二次 hit
3. 继续推进 gateway 侧工程化能力：
   - 限流
   - 统一错误处理
   - 统一观测与追踪
4. 后续再视需要决定是否引入：
   - 更细粒度 cache 策略
   - 文档级失效策略
   - gRPC（当前仍不优先）
