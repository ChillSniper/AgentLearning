# PartC

## 1. PGVector 和 PostgreSQL

PGVector 是什么：

- PostgreSQL 的一个扩展插件，用于存储和检索高维向量（embeddings）
- 支持向量相似度搜索（余弦相似度、欧氏距离等）
- 用于 RAG（检索增强生成）系统中存储文档的向量表示

在你项目中的作用：
代码文档 → 文本分块 → 调用 embedding 模型 → 生成向量 → 存入 PGVector
用户提问 → 生成问题向量 → PGVector 相似度检索 → 返回最相关的代码片段/文档

PostgreSQL 面试常见问题：

- 索引类型：B-Tree, Hash, GiST, GIN（你用的 PGVector 基于 HNSW/IVFFlat 索引）
- 事务隔离级别：Read Uncommitted/Committed, Repeatable Read, Serializable
- MVCC 机制：多版本并发控制如何工作
- 性能优化：EXPLAIN ANALYZE, 慢查询优化, 连接池
- 与 MySQL 区别：更强的 ACID 保证、支持复杂数据类型、更好的扩展性

针对 PGVector 可能被问：

- Q: "为什么选择 PGVector 而不是专用向量数据库(如 Pinecone/Milvus)？"
- A: "考虑到我们项目规模和团队技术栈，PGVector 有几个优势：1)
无需引入额外组件，降低运维成本；2) 可以在同一事务中操作向量和关系数据；3)
对于中小规模(百万级向量)性能足够；4) 团队对 PostgreSQL 更熟悉"

---

## 2. "工程代码+文档" 的分片向量化索引

含义拆解：

工程代码+文档：

- 代码文件：.py, .js, .java 等源代码
- 文档：README.md, API 文档, 设计文档, 注释

分片（Chunking）：
// 示例：长文档分片策略
原始文档(5000字) →
Chunk 1 (500字, overlap 50字)
Chunk 2 (500字, overlap 50字)
Chunk 3 (500字, overlap 50字)
...

为什么要分片：

- Embedding 模型有 token 限制（如 text-embedding-ada-002 限制 8191 tokens）
- 小块检索更精准（避免无关信息干扰）
- 降低向量维度计算成本

向量化索引：

```java
-- PGVector 表结构示例
CREATE TABLE code_embeddings (
id SERIAL PRIMARY KEY,
file_path TEXT,
chunk_text TEXT,
chunk_index INT,
embedding vector(1536),  -- OpenAI ada-002 生成 1536 维向量
metadata JSONB
);
```

-- 创建向量索引（HNSW 或 IVFFlat）
CREATE INDEX ON code_embeddings USING hnsw (embedding vector_cosine_ops);

面试可能问：

- Q: "分片大小如何确定？"
- A: "我们采用 500-1000 token 的固定窗口，overlap
10-20%。对于代码采用语法感知分片（按函数/类边界），对于文档按段落分片，避免语义割裂"

---

## 3. Advisor 记忆组件 + Redis

Advisor 记忆组件是什么：

- AI Agent 的"大脑"，存储对话历史和上下文信息
- 分为短期记忆（当前会话）和长期记忆（跨会话知识）

Redis 的作用：

短期记忆（Session Memory）：

```java
// Redis 结构示例
session:user_123:conversation = [
{role: "user", content: "帮我写一个登录接口", timestamp: ...},
{role: "assistant", content: "好的，需要...", timestamp: ...},
{role: "user", content: "加上 JWT 认证", timestamp: ...}
]

```

// 设置过期时间
EXPIRE session:user_123:conversation 3600

长短期记忆分层：
短期记忆(Redis) ← 最近 N 轮对话
    ↓ 定期抽取关键信息
长期记忆(PostgreSQL) ← 用户偏好、历史任务摘要

Redis 面试高频问题：

- 数据结构：String, Hash, List, Set, ZSet（你可能用了 List 存对话，ZSet 存时序消息）
- 持久化：RDB vs AOF（下面详细说）
- 过期策略：定期删除 + 惰性删除
- 缓存淘汰：LRU, LFU, TTL
- 为什么用 Redis：内存存储快速读写、支持丰富数据结构、支持 TTL 自动过期

针对记忆组件可能被问：

- Q: "如何防止 Redis 内存溢出？"
- A:

1. 为每个会话设置 TTL；
2. 配置 maxmemory-policy 为 allkeys-lru；
3. 定期将冷数据迁移到
PostgreSQL；
4. 监控内存使用率，设置告警"

---

## 4. 流式输出（Streaming）

是什么：

- LLM 生成响应时，逐字/逐 token 返回，而非等待全部生成完毕
- 类似 ChatGPT 打字机效果

技术实现（OpenAI API 示例）：

### 非流式

```java
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "写个函数"}]
)
print(response.choices[0].message.content)  # 一次性返回
```

### 流式

```java
stream = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "写个函数"}],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")  # 逐字输出
```

前端实现（SSE）：

```js
// Server-Sent Events
const eventSource = new EventSource('/api/chat/stream');
eventSource.onmessage = (event) => {
const token = JSON.parse(event.data).content;
appendToChat(token);  // 追加到聊天界面
};
```

优势：

- 用户体验好：立即看到响应（降低感知延迟）
- 可提前取消：生成一半觉得不对立即停止
- 降低超时风险：长响应分批传输

面试可能问：

- Q: "流式输出有什么挑战？"
- A: "1) 前端状态管理复杂（要处理部分内容）；2) 错误处理困难（流中断如何恢复）；3)
无法在响应完成前做后处理（如敏感词过滤需要完整文本）；4) 需要保持长连接（占用服务器资源）"

---

## 5.长短期记忆滚动窗口方案

设计思路：

滚动窗口示例：
固定窗口大小 = 10 轮对话（约 4000 tokens）

对话历史：[Q1-A1, Q2-A2, ..., Q12-A12]
                    ↓
滚动窗口：[Q3-A3, Q4-A4, ..., Q12-A12] ← 保留最近 10 轮
        ↑                        ↑
    最早消息（丢弃Q1-A1, Q2-A2）   最新消息

具体实现：

```python
class MemoryManager:
    def __init__(self, max_tokens=4000, max_rounds=10):
        self.short_term = []  # Redis 存储
        self.long_term = []   # PostgreSQL 存储
        self.max_tokens = max_tokens
        self.max_rounds = max_rounds

    def add_message(self, message):
        self.short_term.append(message)

        # 窗口满时执行滚动
        if len(self.short_term) > self.max_rounds:
            # 策略1: 简单FIFO
            old_msg = self.short_term.pop(0)

            # 策略2: 智能压缩（保留摘要）
            summary = self.summarize(old_msg)
            self.long_term.append(summary)

    def get_context_for_llm(self):
        # 构建发送给 LLM 的上下文
        context = [
            {"role": "system", "content": "你是一个编程助手"},
            *self.get_long_term_summary(),  # 历史摘要
            *self.short_term  # 最近对话
        ]
        return context
```

优化策略：

- 基于重要性滚动：保留用户明确指令，删除闲聊
- 基于 token 计数：动态调整窗口大小，而非固定轮数
- 摘要压缩：旧对话用 LLM 生成摘要存入长期记忆

面试可能问：

- Q: "如何平衡记忆容量和上下文质量？"
- A: "我们采用混合策略：1) 近期 10 轮完整保留；2) 中期(10-50轮)用摘要压缩；3)
远期只保留用户显式保存的知识点；4) 动态根据任务复杂度调整窗口大小"

---

## 6. Redis 持久化对话快照

这里的"持久化"不是指 RDB/AOF！

你简历中的"Redis 定期持久化对话快照"是指：
Redis(内存) → 定期导出 → PostgreSQL(磁盘)

实现方案：

### 方案1: 定时任务（Cron）

```java
@scheduled_task(interval=300)  # 每5分钟
async def persist_conversations():
    # 从 Redis 读取所有活跃会话
    sessions = redis.keys("session:*:conversation")

    for session_key in sessions:
        conversation = redis.lrange(session_key, 0, -1)

        # 保存到 PostgreSQL
        await db.execute("""
            INSERT INTO conversation_snapshots
            (session_id, messages, snapshot_time)
            VALUES ($1, $2, $3)
        """, session_key, json.dumps(conversation), datetime.now())
```

### 方案2: 事件触发（对话结束时）

```java
async def on_conversation_end(session_id):
    conversation = redis.lrange(f"session:{session_id}", 0, -1)
    await save_to_postgres(conversation)
    redis.delete(f"session:{session_id}")
```

如果面试官问 RDB vs AOF：

这是 Redis 自身的持久化机制（你可以说"我们启用了 Redis 的 AOF 模式作为兜底保障"）：

回答示例：
"我们的 Redis 开启了 AOF everysec 模式，每秒同步一次，保证最多丢失1秒数据。同时应用层每 5
分钟将活跃会话导出到 PostgreSQL，这样即使 Redis 完全宕机，也能从 PostgreSQL 恢复到 5
分钟前的状态"

---

### 1. Token 超限和上下文丢失

Token 超限

是什么：

- LLM 有最大输入限制（如 GPT-3.5: 4K tokens, GPT-4: 8K/32K, Claude: 100K）
- 1 token ≈ 0.75 个英文单词，中文约 1-2 个字符

问题场景：
长任务对话：
User: 帮我重构这个5000行的代码
AI: [读取代码 3000 tokens]
User: 再加个功能
AI: [之前代码 + 新需求 + 历史对话 = 超过 4K tokens] ❌ 报错

你的解决方案：

- 滚动窗口限制上下文
- Redis 持久化，允许分阶段加载历史

上下文丢失

是什么：
理想情况：
Q1: 写个用户表
A1: [生成代码]
Q2: 再加个索引  ← AI 记得 Q1 说的是"用户表"

上下文丢失：
Q2: 再加个索引  ← AI 不知道给哪个表加，因为 Q1 被删除了

原因：

- 滚动窗口删除了关键信息
- 会话过期（Redis TTL 到期）
- 系统重启

你的解决方案：

### 关键信息抽取

```python
def extract_context(conversation):
    # 用 LLM 提取关键实体
    summary = llm.generate(f"""
    从以下对话中提取关键信息：
    {conversation}

    输出格式：
    - 当前任务：...
    - 相关文件：...
    - 技术栈：...
    """)
    return summary

# 每次发送请求时携带摘要
context = {
    "conversation_summary": long_term_summary,  # 历史摘要
    "recent_messages": short_term_window        # 最近对话
}
```

面试可能问：

Q: "除了滚动窗口，还有其他办法解决 Token 超限吗？"

A: "有几种方案：

1. 动态摘要：用 LLM 压缩历史对话（map-reduce 模式）
2. RAG检索：不是全部加载历史，而是检索相关历史片段
3. 多轮规划：长任务拆解为子任务，每个子任务独立上下文
4. 模型升级：使用 Claude-3(100K) 或 GPT-4-turbo(128K) 大窗口模型
5. Function Calling：将部分上下文放到工具调用参数中，避免占用对话窗口"

Q: "如何验证你的方案有效？"

A: "我们做了压测：

- 模拟 50 轮对话的长任务（约 20K tokens）
- 监控指标：Token 使用率、响应延迟、任务完成度
- 对比方案：无窗口(直接报错) vs 固定窗口 vs 智能压缩
- 结果：智能压缩方案在保持 95% 任务完成度的同时，Token 使用减少 60%"

---
面试准备建议

必问问题预演：

1. 为什么选这个技术栈？
准备对比表（Redis vs Memcached, PostgreSQL vs MySQL, PGVector vs Pinecone）
2. 遇到过什么坑？
例如："初期用固定滚动窗口，导致重要上下文丢失，后来改成基于重要性评分的淘汰策略"
3. 性能指标？
准备数据："平均响应时间 800ms, P99 延迟 2s, 支持 100 并发用户"
4. 如果重新设计会怎么做？
展示你的思考："现在看来，对于超长上下文任务可以引入 Anthropic Claude 的 100K
窗口，减少压缩损失"

准备一个完整的架构图：

```python
[用户提问]
    ↓
[FastAPI Server]
    ↓
[Redis - 短期记忆] ← 滚动窗口
    ↓
[PostgreSQL - 长期记忆 & PGVector]
    ↓
[LLM (GPT-4)]
    ↓
[流式返回给用户]
```

准备好这些，你就能自信应对技术面试了！有具体哪个点需要深入吗？
