# LangGraph 是什么
LangGraph是由LangChain团队开发的一个低层级Agent编排框架，专为构建状态、长时间运行的AI工作流而设计。与传统的线性LLM不同LangGraph将工作流建模为有向图

- 节点 Node ：执行具体操作的函数，如调用LLM、执行工具、处理数据
- 边 Edge：定义节点之间的流转路径、支持条件分支
- 状态 State：在整个工作流中共享并传递数据

LangGraph是一个基于图Graph的框架，用于构建有状态、可控、可持续运行的LLM应用程序

- 基于图的编排：将复杂流程表示未节点和边
- 有状态执行:在的执行过程中维护和更新状态
- 可控与可观察：支持中断、重试、回滚、人工干预等
- 持久化与恢复：可保持执行状态从中断处恢复
- 高度可扩展：轻松集成工具、检索记忆等能力

<img decoding="async" src="https://www.runoob.com/wp-content/uploads/2026/03/3476ec11-af44-4b42-99a9-8942ffb9c5d3.webp">

# 核心概念

- Graph 图：是整个工作流的蓝图，定义个Agent的完整逻辑，它有节点Nodes 和 边 Edges 组成
```shell
StateGraph
    |-- Nodes 节点
    |    |-- node_a
    |    |-- node_b
    |    |__ node_c
    |-- Edges 边
         |-- START -> node_a
         |-- node_a -> node_b 条件边
         |-- node_a -> node_c 条件边
         |__ node_b -> END 
```

- State 状态：是贯穿整个图的共享数据结构，每个节点可以读取和更新State，更新后的State会传递给下一个节点
```python
from typing import TypedDict, Annotated
from langgraph.graph import add_messages

class MyState(TypedDict):
    messages: Annotated[list, add_messages]  # 消息列表（自动追加）
    user_name: str                            # 用户名称
    step_count: int                           # 步骤计数
```
重要：Annotated[list, add_messages] 表示该字段使用 add_messages 作为 reducer——新消息会追加到列表而不是覆盖。这是 LangGraph 状态管理的核心机制

```shell

state    ---->  NodeA  ----> state    ----> Node B   ----> State      ----> END
{msg:[]}       处理输入       {msg:[A]}      生成响应       {msg:[A,B]}

```

- Nodes 节点：节点是普通的Python函数接收当前的state，返回更新后的state
```python
def my_node(state: MyState) -> dict:
    # 读取状态
    messages = state["messages"]
    
    # 执行操作...
    result = "处理结果"
    
    # 返回更新的字段（不需要返回所有字段）
    return {"messages": [{"role": "ai", "content": result}]}
```
- Edge 边：定义节点间的流转方式
```python
 1 普通边：固定路径，node_a -> node_b
 2 条件边：根据state动态路由，node_a -> node_b Or node_c
 3 起始边: START -> 第一个节点
 4 结束边: 某节点 -> END
```
