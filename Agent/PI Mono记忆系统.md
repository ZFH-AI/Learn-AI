# PI 记忆系统：持久化、压缩与恢复

PI 的记忆系统用于解决 AI coding assistant 的核心问题：对话会越来越长，模型上下文窗口有限，但用户希望工具能继续理解之前的目标、约束、文件修改、分支尝试和当前进度。

PI 的做法不是把旧历史直接删除，而是把完整历史持久化到 session 文件中，并在发送给 LLM 前重建一个更适合当前任务的上下文：

- 历史完整保存在 append-only JSONL 中，支持审计、恢复和分支。
- LLM 看到的是摘要、分支上下文和近期消息的组合。
- 压缩只改变“恢复方式”，不改变底层历史事实。

## 1. 总体架构

PI 的记忆系统可以理解为一个 **session-entry 驱动的持久化上下文系统**。

核心事实源是 session JSONL 文件。每一行都是一个 entry，entry 之间通过 `id` / `parentId` 组成树。当前对话位置由 active leaf 表示。恢复上下文时，PI 从 leaf 往 root 回溯，取出当前路径上的 entries，再把 compaction 和 branch summary 转换为 LLM 可见的摘要消息。

```text
用户输入 / 工具结果 / 模型回复
          │
          ▼
   SessionEntry 追加写入
          │
          ▼
 append-only tree JSONL
          │
          ├─ /resume 恢复会话
          ├─ /tree 切换分支
          ├─ /compact 压缩上下文
          └─ 自动压缩与 overflow 恢复
          │
          ▼
  重建 LLM 上下文
          │
          ├─ branchSummary
          ├─ compactionSummary
          └─ recent messages
          │
          ▼
        LLM 继续工作
```

常见 entry 类型：

| Entry 类型 | 用途 |
|---|---|
| `message` | 用户、助手、工具结果、自定义消息等普通对话内容 |
| `compaction` | 压缩摘要，以及从哪里开始保留近期消息 |
| `branch_summary` | 从一个分支切到另一个分支时，对离开分支的摘要 |
| `model_change` | 会话中模型切换记录 |
| `thinking_level_change` | 推理强度切换记录 |
| `custom` | 扩展持久化的自定义状态 |
| `session_info` | 会话相关元数据 |

## 2. 持久化：append-only tree JSONL

PI 的 session 默认保存到：

```text
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

其中 `<path>` 是工作目录路径的编码形式。每个 session 文件第一行是 header，后续每一行是一个 entry。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"Hello"}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}]}}
```

这个设计有三个关键点。

第一，历史只追加，不覆盖。压缩不会删除旧消息，只会追加一条新的 `compaction` entry。分支切换也不会改写历史，而是在树上移动 leaf，并可追加 `branch_summary`。

第二，树结构支持原地分支。用户可以通过 `/tree` 回到任意历史点继续对话，新消息会挂到被选择节点之后，形成新分支。

第三，恢复时只读取当前路径。JSONL 中可以有多个分支，但 LLM 上下文只由当前 active leaf 到 root 的路径决定。

```text
session-abc123 (root)
    │
    └─ msg-001 (初始 user message)
        │
        └─ msg-002 (assistant response)
            │
            ├─ msg-003 (分支 A：方案 1)
            │   │
            │   └─ msg-005 (assistant 在分支 A 的回复)
            │       │
            │       └─ msg-006 (user 追问) ← leafId_A
            │
            └─ msg-004 (分支 B：方案 2)
                │
                └─ msg-007 (assistant 在分支 B 的回复) ← leafId_B
```

## 3. 压缩：摘要替代旧上下文

LLM 上下文窗口有限。当上下文接近模型限制时，PI 会把旧消息总结成结构化摘要，同时完整保留近期消息。

自动压缩触发条件是：

```text
contextTokens > contextWindow - reserveTokens
```

默认配置：

| 配置项 | 默认值 | 含义 |
|---|---:|---|
| `enabled` | `true` | 是否启用自动压缩 |
| `reserveTokens` | `16384` | 为下一次回复预留的 token |
| `keepRecentTokens` | `20000` | 压缩后仍完整保留的近期 token |

也可以用 `/compact [instructions]` 手动触发，并通过 instructions 指定摘要重点。

### 3.1 压缩前后

压缩时，PI 会找到一个安全的 cut point，把更早的消息总结为 `summary`，并记录 `firstKeptEntryId`。之后恢复上下文时，LLM 看到的是 summary 加上 `firstKeptEntryId` 之后的近期消息。

```text
Before compaction:

  entry:  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┐
        │ hdr │ usr │ ass │ tool│ usr  │ ass │ tool│ tool │ ass  │ tool│
        └─────┴─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┘
                └────────┬──────┘ └──────────────┬──────────────┘
               messagesToSummarize            kept messages
                                   ↑
                          firstKeptEntryId (entry 4)

After compaction (new entry appended):

  entry:  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool│ usr  │ ass │ tool│ tool │ ass  │ tool│ cmp │
        └─────┴─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
               └──────────┬─────┘ └──────────────────────┬───────────────────┘
                 not sent to LLM                    sent to LLM
                                                         ↑
                                              starts from firstKeptEntryId

What the LLM sees:

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    prompt   from cmp          messages from firstKeptEntryId
```

底层 JSONL 中，旧消息仍然存在；只是发送给 LLM 时不再直接展开。

### 3.2 安全切割

PI 只在语义安全的位置切割：

- `user` message
- `assistant` message
- `bashExecution` message
- `custom` message
- `branch_summary` message
- `compactionSummary` message

不能在 `toolResult` 上切割，因为 tool result 必须和前面的 tool call 配对。如果保留了工具结果却压缩掉工具调用，LLM 会看到一个没有来源的结果，语义不完整。

### 3.3 Split Turn

有时单个 turn 太大，超过 `keepRecentTokens`。这时 cut point 可能落在一个 turn 内部，PI 会把这个 turn 的前缀单独总结，保留后缀原文。

```text
Split turn (one huge turn exceeds budget):

  entry:  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── kept (7-8)

  isSplitTurn = true
  messagesToSummarize = []  (no complete turns before)
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

这种情况下会生成两类摘要：

- 历史摘要：描述之前的上下文。
- turn prefix 摘要：描述当前大 turn 的早期部分，让 LLM 能理解保留的后缀。

## 4. 摘要格式：结构化 checkpoint

PI 的 compaction 和 branch summarization 使用同一类结构化摘要格式。

摘要生成使用专门的 system prompt。它把模型限定为“上下文摘要助手”，明确要求模型只输出结构化摘要，不继续对话、不回答被摘要对话中的问题。

```text
You are a context summarization assistant. Your task is to read a conversation between a user and an AI coding assistant, then produce a structured summary following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the conversation. ONLY output the structured summary.
```

这个约束很重要。待压缩内容本身是用户和 coding assistant 的历史对话，如果不明确禁止继续对话，模型可能把其中某个用户问题当成当前请求来回答，而不是生成 checkpoint。

### 4.1 首次压缩提示词

首次压缩没有 previous summary，PI 要求模型从待压缩对话中生成完整 checkpoint：

```markdown
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
- [Requirements mentioned by user]

## Progress
### Done
- [x] [Completed tasks]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues, if any]

## Key Decisions
- **[Decision]**: [Rationale]

## Next Steps
1. [What should happen next]

## Critical Context
- [Data needed to continue]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

提示词还要求：

- 使用完全相同的 section 结构。
- 每个 section 保持简洁。
- 保留精确的文件路径、函数名和错误信息。
- 如果没有约束或关键上下文，用 `(none)` 明确表示。

这个格式的重点不是压缩成最短文本，而是把“继续工作需要的状态”稳定保存下来：

- 用户目标是什么。
- 用户明确说过哪些偏好和约束。
- 已完成、正在做、被阻塞的事项分别是什么。
- 做过哪些关键决策，原因是什么。
- 下一步应该做什么。
- 哪些文件被读过或改过。

### 4.2 增量压缩提示词

重复压缩时，PI 会把上一次 summary 作为 `previousSummary` 输入，让新摘要做增量更新，而不是从零生成。

增量压缩的输入包含两块：

```text
<conversation>
[new messages to summarize]
</conversation>

<previous-summary>
[existing structured summary]
</previous-summary>
```

增量更新规则：

- 保留 previous summary 中已有的所有信息。
- 只添加新消息支持的新进展、新决策和新上下文。
- 当任务完成时，把 `In Progress` 中的事项移动到 `Done`。
- 更新 `Next Steps`，反映最新计划。
- 保持文件路径、函数名、错误信息精确。
- 不重复 previous summary 中已经存在的信息。
- 输出完整更新后的 summary，不输出 diff。

首次压缩和增量压缩的差异：

| 维度 | 首次压缩 | 增量压缩 |
|---|---|---|
| 输入 | 待压缩对话 | 新对话 + `<previous-summary>` |
| 输出 | 新的完整摘要 | 更新后的完整摘要 |
| 重点 | 从头提取状态 | 保留旧状态并合并新状态 |
| Progress | 重新分类 | 完成项从 `In Progress` 移到 `Done` |
| Next Steps | 从当前对话生成 | 更新为最新计划 |

示例：

```text
<previous-summary>
## Goal
用户请求修复登录 bug

## Constraints & Preferences
- 使用 TypeScript
- 保持向后兼容

## Progress
### Done
- [x] 分析问题根因（config.ts 第 50 行类型不匹配）

### In Progress
- [ ] 修复 config.ts 的类型问题

### Blocked
(none)

## Key Decisions
- **使用类型断言修复**：因为运行时类型正确，只是静态类型检查不通过

## Next Steps
1. 修改 config.ts 第 50 行
2. 运行测试验证

## Critical Context
- config.ts 第 50 行需要修复类型问题
</previous-summary>

<conversation>
[User]: 继续修复
[Assistant]: 已修改 config.ts 第 50 行，现在运行测试
[Assistant tool calls]: bash(command="npm test")
[Tool result]: all tests passed
</conversation>
```

更新后的摘要会保留旧信息，并把修复和测试验证移动到 `Done`：

```text
## Progress
### Done
- [x] 分析问题根因（config.ts 第 50 行类型不匹配）
- [x] 修复 config.ts 的类型问题
- [x] 运行测试验证

### In Progress
(none)
```

### 4.3 Split turn 前缀提示词

当单个 turn 太大而被切成 prefix 和 suffix 时，PI 会对 prefix 使用专门提示词。目标不是生成完整 session checkpoint，而是让 LLM 能理解保留下来的 suffix。

输出格式：

```markdown
## Original Request
[What did the user ask for in this turn?]

## Early Progress
- [Key decisions and work done in the prefix]

## Context for Suffix
- [Information needed to understand the retained recent work]
```

这个摘要会和保留的 suffix 一起进入上下文，避免出现“近期消息还在，但早期请求和早期工具调用被压缩后语义断裂”的问题。

### 4.4 输入序列化与自定义重点

生成摘要前，PI 会先把内部 `AgentMessage[]` 转成 LLM message，再序列化为纯文本并包进 `<conversation>...</conversation>`：

```text
[User]: What they said
[Assistant thinking]: Internal reasoning
[Assistant]: Response text
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: Output from tool
```

序列化有两个作用：

- 防止模型把历史对话误判为当前要继续执行的对话。
- 把 tool call、tool result、thinking content 统一成适合摘要模型读取的文本。

tool result 在摘要序列化时最多保留 2000 字符，超出部分用截断标记替代，避免大文件输出或长命令输出吞掉摘要预算。

手动 `/compact [instructions]` 或 API 传入的 `customInstructions` 会追加到 base prompt 后：

```text
Additional focus: [custom instructions]
```

压缩调用还支持 `thinkingLevel`，可在需要更高质量摘要时启用更强推理。摘要最大输出 token 通常按预留窗口的一部分控制，避免摘要本身占用过多上下文预算。

## 5. 文件操作跟踪

对 coding agent 来说，只记住自然语言对话不够。PI 还会在 compaction 和 branch summary 中累计文件操作信息。

默认记录两类文件：

| 字段 | 含义 |
|---|---|
| `readFiles` | 被读取但未修改的文件 |
| `modifiedFiles` | 被写入或编辑过的文件 |

计算规则：

```text
modifiedFiles = written + edited
readFiles = read - modifiedFiles
```

这些信息会写入 entry 的 `details` 字段，也会附加到摘要文本中。下一次压缩时，PI 会继承并合并之前的文件列表，从而在长对话中保留持续的文件上下文。

## 6. 分支摘要：切换路径时保留离开分支的上下文

PI 的 session 是树，不是线性日志。用户通过 `/tree` 切到另一个历史点继续工作时，当前分支上的尝试可能仍然有价值。PI 可以把即将离开的分支总结成 `branch_summary`，挂到新位置附近。

```text
Tree before navigation:

         ┌─ B ─ C ─ D (old leaf, being abandoned)
    A ───┤
         └─ E ─ F (target)

Common ancestor: A
Entries to summarize: B, C, D

After navigation with summary:

         ┌─ B ─ C ─ D ─ [summary of B,C,D]
    A ───┤
         └─ E ─ F (new leaf)
```

分支摘要流程：

```text
用户在 /tree 中切换到另一个分支
  │
  ├─ 找到 old leaf 与 target 的 common ancestor
  ├─ 收集 old leaf 到 common ancestor 之间的 entries
  ├─ 提取 readFiles / modifiedFiles
  ├─ 调用 LLM 生成结构化摘要
  ├─ 追加 BranchSummaryEntry
  └─ 把 leaf 移动到目标分支
```

这样，LLM 不需要重放旧分支的完整对话，也能知道离开分支中做过什么、读改了哪些文件、哪些结论可能影响当前路线。

## 7. 恢复：从完整历史重建可用上下文

恢复 session 时，PI 做的不是简单读取所有消息，而是从树和压缩信息中重建当前 LLM 应该看到的上下文。

```text
1. 读取 JSONL 文件
   └─ 建立 id → entry 的索引

2. 从 active leaf 向 root 回溯
   └─ 得到当前分支路径上的 entries

3. 处理 compaction entries
   ├─ 转换为 compactionSummary message
   ├─ 读取 firstKeptEntryId
   └─ 从 firstKeptEntryId 开始保留 recent messages

4. 处理 branch_summary entries
   └─ 转换为 BranchSummaryMessage

5. 构建最终 messages
   └─ branchSummary + compactionSummary + recent messages
```

示例：

```text
msg-001 (user1)
msg-002 (assistant1)
brs-001 (branch_summary for branch A)
cmp-001 (compaction: firstKeptEntryId=msg-010, summary="...")
msg-010 (user2)
msg-011 (assistant2)
msg-012 (user3)
msg-013 (assistant3) ← current leaf
```

恢复后 LLM 看到：

```text
[branchSummaryMessage: "分支 A 中完成了..."]
[compactionSummaryMessage: "## Goal\n...\n## Progress\n..."]
[msg-010: user2]
[msg-011: assistant2]
[msg-012: user3]
[msg-013: assistant3]
```

这个模型同时满足三个目标：

- 旧历史可追溯：JSONL 中仍有完整消息。
- 当前上下文不超限：LLM 不直接接收所有旧消息。
- 继续工作有状态：摘要保留目标、约束、进度、文件和决策。

## 8. 扩展点

PI 允许扩展介入 compaction 和 tree navigation。

`session_before_compact` 可以在自动压缩或 `/compact` 前触发，扩展可以：

- 取消压缩。
- 使用自定义模型生成摘要。
- 提供自定义 `details`。
- 根据业务需求改变摘要内容。

`session_before_tree` 可以在 `/tree` 导航前触发，扩展可以：

- 取消导航。
- 在用户选择生成分支摘要时提供自定义摘要。
- 在 `details` 中保存扩展状态。

`CompactionEntry` 和 `BranchSummaryEntry` 都支持泛型 `details` 字段。内建实现默认存储文件列表，扩展可以存储自己的 JSON-serializable 数据。

## 9. 设计取舍

PI 的记忆系统做了几个明确取舍。

| 取舍 | PI 的选择 | 原因 |
|---|---|---|
| 历史是否删除 | 不删除，只追加 | 支持审计、恢复、调试和分支 |
| 压缩保存在哪里 | 新增 `compaction` entry | 压缩本身也是可追溯事件 |
| 分支如何表达 | `parentId` tree + active leaf | 支持原地探索多条路线 |
| LLM 看什么 | 摘要 + 分支摘要 + 近期消息 | 控制 token，同时保留工作状态 |
| 文件上下文如何保留 | `details` + 摘要中的 file list | 让 coding agent 知道读改过什么 |
| 扩展如何参与 | event hook + custom details | 允许替换摘要策略，不破坏核心格式 |

## 10. 总结

PI 的记忆系统不是一个简单的聊天记录保存功能，而是一套面向长程 coding agent 的上下文管理机制。

它的核心是：

- 用 append-only tree JSONL 保存完整历史。
- 用 compaction 把旧上下文转换为结构化 checkpoint。
- 用 branch summary 保留离开分支的有效信息。
- 用 `firstKeptEntryId` 把摘要和近期原文连接起来。
- 用 `details` 累计文件操作和扩展状态。

最终效果是：PI 可以在长对话、多分支和上下文窗口限制之间取得平衡，让 LLM 看到足够继续工作的状态，同时保留完整历史用于恢复、审计和扩展。
