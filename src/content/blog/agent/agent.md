---
title: 带你深入了解Agent是如何工作的
link: agent
catalog: true
date: 2025-01-05 13:00:00
description: agent是什么？如何实现agent
tags:
  - python
  - agent
categories:
  - [Agent]
---

## 1. 什么是 Agent

这是面试中经常会被问到的问题——"你说你做 Agent，那什么是 Agent？" 很少有人能回答得明白。

简单来说：**Agent = `LLM` + `工具调用` + `规划能力` + `记忆管理`**。

它不再是一个只能"一问一答"的聊天机器人，而是能够 **`端到端地帮助人完成一个复杂任务`** 的智能体。

### 1.1 从 LLM 到单 Agent

传统的 LLM 交互是这样的：

```plain
用户提问 → LLM 回答 → 结束
```

但现实中的任务往往更复杂，比如"帮我查一下今天的天气，然后写一封邮件告诉老板"。单纯的 LLM 做不到——它不能访问天气 API，也不能发邮件。

**Agent 的出现就是为了解决这个问题**：让 LLM 具备调用外部工具的能力，从"问答"升级为"执行"。

```plain
用户提出复杂任务
    → Agent 规划步骤
    → 调用工具 A（查天气）
    → 调用工具 B（发邮件）
    → 汇总结果返回用户
```

### 1.2 从单 Agent 到多 Agent

当任务更加复杂时，一个 Agent 可能力不从心。这时可以引入 **`多 Agent 协作`**：

- **`Agent A`**：负责信息收集（搜索、爬虫）
- **`Agent B`**：负责数据分析（计算、统计）
- **`Agent C`**：负责内容生成（写报告、邮件）

多个 Agent 之间通过消息传递协作，就像一个小团队各司其职。

### 1.3 Agent 架构解析

一个完整的 Agent 系统通常由以下核心组件组成：

| 组件 | 作用 | 类比 |
|------|------|------|
| **LLM（大脑）** | 理解意图、做出决策 | 人的大脑 |
| **Tools（工具）** | 执行具体操作（查天气、搜索、发邮件等） | 人的手 |
| **Planning（规划）** | 将复杂任务拆解为多个步骤 | 人的思考过程 |
| **Memory（记忆）** | 保存对话上下文和历史信息 | 人的记忆 |

---

## 2. Function Call 深度理解

`Function Call`（函数调用）是 Agent 的**核心能力**——它是 LLM 连接外部世界的桥梁。

理解 Function Call，要从四个层次递进：

1. 单个工具的调用
2. 多个工具的选择调用
3. 多轮链式工具调用
4. 单轮多工具并行调用

### 2.1 单个工具调用

这是最基础的场景：LLM 判断需要调用某个工具，调用后将结果返回给用户。

#### LLM 的"幻觉"问题

先看一个**没有工具**时的例子——直接问 LLM "北京今天天气如何"：

```python
client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

completion = client.chat.completions.create(
    model="qwen-plus",
    messages=[
        {'role': 'system', 'content': 'You are a helpful assistant.'},
        {'role': 'user', 'content': '北京今天天气如何'}
    ],
    stream=True,
)
```

LLM 会**编造一个看似合理的回答**，但这并不是真实数据！这就是"`幻觉`"。

#### 加入工具后

有了 Function Call，LLM 不再自己编造，而是**识别出需要查天气**，然后请求调用我们定义的工具：

```python
# 定义工具列表
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_current_weather",
            "description": "当你想查询指定城市的天气时非常有用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "城市或县区，比如北京市、杭州市、余杭区等。",
                    }
                },
                "required": ["location"],
            },
        },
    },
]

# 模拟天气查询工具
def get_current_weather(arguments):
    weather_conditions = ["晴天", "多云", "雨天", "大风", "下雪"]
    random_weather = random.choice(weather_conditions)
    location = arguments.get("location", "未知地区")
    return f"{location}今天是{random_weather}。"
```


#### 为什么加个 tool list 就能实现工具调用？

本质上有两种实现方式：

**方式一：原生 API 的 tools 参数（自动挡）**

直接在 API 调用时传入 `tools` 参数，模型内部已经学会了如何解析和选择工具。这是最推荐的方式。

**方式二：通过 System Prompt 注入工具描述（手动挡）**

将工具信息写入 `System Prompt`，让模型以 XML 格式输出调用意图，再由 Python 代码解析执行：

```python
system_prompt = f"""你是一个智能助手...

# 工具列表 (Tools)
<tools>
{tools_content}
</tools>

# 输出格式要求
如果需要调用工具，请严格按照以下 XML 格式返回：
<tool_call>
{{"name": "函数名称", "arguments": {{"参数名": "参数值"}}}}
</tool_call>
"""

# 手动解析 XML
match = re.search(r"<tool_call>(.*?)</tool_call>", content, re.DOTALL)
if match:
    tool_call_data = json.loads(match.group(1).strip())
    func_name = tool_call_data.get("name")
    arguments = tool_call_data.get("arguments")
    # 手动执行并拼接结果...
```

手动挡需要三步操作：**`手动解析` → `手动执行` → `手动拼接历史`**。理解这个过程有助于深入理解 Function Call 的本质。

---

## 3. Agent 系统设计

了解了 Function Call 之后，我们来看更高层的 Agent 系统设计模式。

### 3.1 Prompt Chain（提示词链）

**核心思想**：把一个大任务拆成多个小任务，每一步的结果流入下一步。

这是一种 **`Simple Workflow`** 模式，不依赖工具调用，而是通过**多阶段 LLM 调用**来完成复杂任务。

#### 以"博客生成"为例

```plain
Step 1 [策略师] 主题分析 → 输出分析报告（JSON）
    ↓
Step 2 [架构师] 大纲生成 → 基于分析生成文章骨架
    ↓
Step 3 [作家] 内容撰写 → 基于骨架填充内容
    ↓
Step 4 [编辑] SEO优化润色 → 最终发布版本（流式输出）
```

每个步骤使用不同的**角色（Persona）**和**强约束 Prompt**：

```python
def run_blog_workflow(user_topic):
    # Step 1: 主题分析（策略师）
    analysis_result = get_completion("1-主题分析", f"""
    请针对主题 "{user_topic}" 进行深度内容策略分析。
    输出 JSON：target_audience, core_keywords, unique_angle, tone...
    """)

    # Step 2: 大纲生成（架构师）—— 使用 Step 1 的输出
    outline_result = get_completion("2-大纲生成", f"""
    基于以下【分析报告】，设计文章大纲。
    【分析报告】：{analysis_result}
    """)

    # Step 3: 内容撰写（作家）—— 使用 Step 2 的输出
    draft_result = get_completion("3-内容撰写", f"""
    基于以下【大纲】撰写完整文章。禁止空洞废话，必须包含案例。
    【大纲】：{outline_result}
    """)

    # Step 4: SEO优化（编辑）—— 流式输出最终结果
    get_stream_completion(f"""
    对以下初稿进行SEO优化和润色。以 Markdown 格式输出。
    【初稿】：{draft_result}
    """)
```

#### Prompt Chain 的`四个核心`设计原则

| 原则 | 说明 |
|------|------|
| **链式传递** | 上一步的输出 = 下一步的输入，形成逻辑流水线 |
| **角色扮演** | 每步设置不同 Persona，让模型只专注本角色的事 |
| **强约束 Prompt** | 每步都有严格的输出要求，防止"AI 味"泛化回答 |
| **混合输出模式** | 中间步骤非流式（用户不关心），最终步骤流式（提升体验） |

### 3.2 ReAct-Reflection（推理-行动-反思）

`ReAct` 模式是 Agent 系统中最经典的架构之一，核心是一个 **"`思考`→`行动`→`观察`→`反思`"的循环**。

```plain
用户问题
  → [Planner] 思考：需要做什么？
  → [Executor] 行动：调用工具
  → [Observer] 观察：查看结果
  → [Reflector] 反思：结果够了吗？需要继续吗？
  → 如果不够 → 回到思考步骤
  → 如果够了 → 生成最终回答
```

以一个物流查询的 Agent 为例，展示 ReAct 的分步推理：

```python
# 模拟数据库
MOCK_DB = {
    "orders": {"123456": {"status": "Shipped", "tracking_id": "SF789", "user_id": "u_007"}},
    "logistics": {"SF789": {"status": "In Transit (Stuck)", "history": [...]}},
    "carrier_internal": {"SF789": {"error_code": "ADDR_INVALID", "note": "地址不完整"}},
    "users": {"u_007": {"name": "James", "address": "杭州西湖区...(缺少楼号)"}}
}
```

Agent 的推理链条：

```plain
用户："我的订单 123456 怎么还没到？"

Round 1 - 思考：需要先查订单状态
         行动：query_order("123456") → 获得物流单号 SF789
         反思：拿到物流单号了，继续查物流

Round 2 - 思考：查看物流轨迹
         行动：query_logistics("SF789") → 发现状态异常 "Stuck"
         反思：物流卡住了，需要查内部系统

Round 3 - 思考：查询内部异常原因
         行动：query_internal_system("SF789") → 地址不完整
         反思：找到原因了，还需要用户信息来给出解决方案

Round 4 - 思考：获取用户地址信息
         行动：query_user_info("u_007") → 地址缺楼号
         反思：信息足够了，可以生成最终回答

最终回答："您的包裹因地址不完整（缺少楼号）被卡在杭州中转站，
         请补充完整地址后我们会立即安排重新派送。"
```

核心代码结构：

```python
class ModularAgent:
    def run(self, query):
        """主控制循环"""
        context = {"query": query, "history": [], "reflections": []}

        for round_num in range(MAX_ROUNDS):
            # 1. Planner 决策
            plan = self.planner(context)

            if plan["decision"] == "finish":
                return self.generate_final_answer(context)

            # 2. Executor 执行工具
            result = self.executor(plan["tool"], plan["arguments"])
            context["history"].append(result)

            # 3. Reflector 反思
            reflection = self.reflector(context)
            context["reflections"].append(reflection)
```

### 3.3 中央调度-执行架构

这种架构将 Agent 系统分为两个明确的角色：

- **`中央调度器`**：负责任务规划、分配和监控
- **`执行器`**：负责执行具体的工具调用

适用于更大规模的多 Agent 系统，调度器可以同时管理多个执行器，实现任务的并行处理。

---

## 4. 短期记忆设计

Agent 的对话上下文是以 `messages` 列表的形式传入 LLM 的。那么这个 messages 该如何存储和管理？

### 整体架构

```plain
持久化存储（MySQL/MongoDB）        Redis（工作区）
      ↓                              ↓
   长期保存所有历史                 当前活跃会话的上下文
```

- **新开对话**：创建新 session → 对话存入 Redis → 聊完持久化到数据库
- **继续历史对话**：从数据库加载到 Redis → 继续聊天 → 聊完更新数据库

### 为什么用 Redis？

| 存储 | 作用 | 特点 |
|------|------|------|
| **Redis** | 活跃会话的工作区 | 读写快（内存级）、支持自动过期 |
| **MySQL/MongoDB** | 所有历史记录的持久化 | 持久可靠、支持复杂查询 |

### 代码实现

使用 `Redis` 的 **List** 数据结构存储对话历史：

```python
import redis
import json
from openai import OpenAI

redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

class RedisChatHistory:
    def __init__(self, session_id, ttl=3600):
        self.session_id = f"chat_history:{session_id}"
        self.ttl = ttl  # 过期时间，默认1小时

    def add_message(self, role, content):
        """添加一条消息到 Redis"""
        message = {"role": role, "content": content}
        redis_client.rpush(self.session_id, json.dumps(message, ensure_ascii=False))
        redis_client.expire(self.session_id, self.ttl)  # 刷新过期时间

    def get_messages(self):
        """获取当前会话的所有历史消息"""
        raw_messages = redis_client.lrange(self.session_id, 0, -1)
        return [json.loads(msg) for msg in raw_messages]

    def clear(self):
        """清空会话历史"""
        redis_client.delete(self.session_id)

    def initialize_if_empty(self, system_prompt):
        """如果是新会话，初始化 System Prompt"""
        if redis_client.llen(self.session_id) == 0:
            self.add_message("system", system_prompt)
```