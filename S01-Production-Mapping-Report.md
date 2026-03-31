# S01 Agent Loop: 从教学理论到产品级实现的映射报告

> 本报告以 [S01-Agent-Loop.md](https://github.com/talonMMA/My-Brain/blob/master/02-Study/AI-ML/Learn-Claude-Code/S01-Agent-Loop.md) 学习笔记中的 6 个核心组件为主线，逐一对应 Claude Code 源码中的产品级实现，揭示"108 行教学代码"如何演变为"数万行工程代码"。

---

## 总览：理论 vs 产品的结构对照

```
S01 教学版（108 行）                    Claude Code 产品版（数万行）
─────────────────────                   ──────────────────────────
agent_loop()                     →      queryLoop()             (src/query.ts)
client.messages.create()         →      queryModelWithStreaming()(src/services/api/claude.ts)
TOOLS = [{ "name": "bash" }]    →      40+ 工具定义              (src/tools/**/*)
run_bash()                       →      runTools() + 分层执行     (src/services/tools/*)
SYSTEM = "You are..."            →      getSystemPrompt() 动态组装(src/constants/prompts.ts)
if __name__ == "__main__": REPL  →      REPL.tsx React 组件       (src/screens/REPL.tsx)
messages = []                    →      normalizeMessagesForAPI() (src/utils/messages.ts)
```

---

## 1. 核心循环：`agent_loop()` → `queryLoop()`

### S01 教学版

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(...)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

**核心骨架**：`while True → 调 API → 检查 stop_reason → 执行工具 → 追加结果 → 继续`

### 产品版：`src/query.ts:241-1729`（~1500 行）

```typescript
async function* queryLoop(params: QueryParams): AsyncGenerator<...> {
  let state: State = { messages: params.messages, ... }

  while (true) {                                          // ← 同样的 while(true)
    let { toolUseContext } = state
    const { messages, ... } = state

    // ====== 调 API 前的准备（教学版没有）======
    messagesForQuery = await applyToolResultBudget(...)   // 工具结果预算裁剪
    messagesForQuery = await deps.microcompact(...)       // 微压缩
    await contextCollapse.applyCollapsesIfNeeded(...)     // 上下文折叠
    const fullSystemPrompt = asSystemPrompt(...)          // 动态构建 system prompt
    await deps.autocompact(...)                           // 自动压缩（可能触发摘要）

    // ====== 调 API（对应教学版的 client.messages.create）======
    const toolUseBlocks: ToolUseBlock[] = []
    let needsFollowUp = false

    for await (const message of deps.callModel({          // 流式 API 调用
      messages: prependUserContext(messagesForQuery, userContext),
      systemPrompt: fullSystemPrompt,
      tools: toolUseContext.options.tools,
      ...
    })) {
      if (message.type === 'assistant') {
        assistantMessages.push(message)
        const msgToolUseBlocks = message.message.content
          .filter(content => content.type === 'tool_use')
        if (msgToolUseBlocks.length > 0) {
          toolUseBlocks.push(...msgToolUseBlocks)
          needsFollowUp = true                            // ← 对应教学版的 stop_reason 检查
        }
      }
    }

    // ====== 退出条件（对应教学版的 if stop_reason != "tool_use": return）======
    if (!needsFollowUp) {
      // ... 错误恢复、stop hooks 等
      return { reason: 'completed' }
    }

    // ====== 执行工具（对应教学版的 for block in response.content）======
    const toolUpdates = streamingToolExecutor
      ? streamingToolExecutor.getRemainingResults()
      : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

    for await (const update of toolUpdates) {
      if (update.message) {
        toolResults.push(
          ...normalizeMessagesForAPI([update.message], ...).filter(...)
        )
      }
    }

    // ====== 追加结果并继续（对应教学版的 messages.append + while 继续）======
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      turnCount: nextTurnCount,
      ...
    }
  } // while (true)
}
```

### 关键差异总结

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **循环退出信号** | `response.stop_reason != "tool_use"` | `needsFollowUp` 布尔标志（注释明确说 stop_reason 不可靠） |
| **API 调用方式** | 同步 `client.messages.create()` | 异步流式 `async generator` — 边流边处理 |
| **工具执行** | 串行逐个执行 | 并行 + 流式穿插（模型还在流，工具已在跑） |
| **上下文管理** | messages 只增不减 | 微压缩 + 自动压缩 + 上下文折叠 + 历史裁剪 |
| **错误恢复** | 无 | 6 种 `continue` 恢复路径（见下文） |
| **状态管理** | 直接修改列表 | 不可变 `State` 对象，每次 `state = next; continue` |
| **返回值** | 无（void） | `AsyncGenerator` yield 每个事件，返回终止原因 |

### 产品版独有的 6 种恢复路径

教学版循环只有一条路：工具调用 → 继续，无工具 → 退出。产品版有 6 种 `state = next; continue` 的恢复/重入点：

```
1. collapse_drain_retry    — 上下文折叠排空后重试（413 错误恢复）
2. reactive_compact_retry  — 响应式压缩后重试（413 错误恢复）
3. max_output_tokens_escalate — 输出 token 上限升级（8k → 64k）
4. max_output_tokens_recovery — 注入 "继续" 指令让模型接续
5. stop_hook_blocking      — stop hook 返回阻塞错误后重试
6. token_budget_continuation — token 预算未用完时注入 nudge 让模型继续
7. next_turn               — 正常工具执行后进入下一轮（对应 S01 教学版）
```

只有第 7 种对应教学版的正常循环路径，前 6 种全是产品级的"自愈"机制。

---

## 2. LLM API 调用：`client.messages.create()` → `queryModelWithStreaming()`

### S01 教学版

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

5 个参数，同步调用，一次返回完整响应。

### 产品版：`src/services/api/claude.ts`（~2400 行）

```typescript
// 入口函数（公共 API）
export async function* queryModelWithStreaming({
  messages, systemPrompt, thinkingConfig, tools, signal, options,
}): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage> {
  yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}

// 实际 API 调用
const result = await anthropic.beta.messages.create(
  { ...params, stream: true },                     // ← 始终流式
  { signal, headers: { ... } },
).withResponse()

// 流式事件处理循环
for await (const part of stream) {
  switch (part.type) {
    case 'message_start':       // 捕获初始 BetaMessage
    case 'content_block_start': // 初始化内容块（text/tool_use/thinking）
    case 'content_block_delta': // 累积增量（文本/JSON/思考）
    case 'content_block_stop':  // 创建 AssistantMessage 并 yield
    case 'message_delta':       // 更新 usage 和 stop_reason
    case 'message_stop':        // 清理
  }
}
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **调用方式** | 同步，等待完整响应 | 流式 AsyncGenerator，逐 token yield |
| **参数数量** | 5 个 | 30+ 个（model, system, tools, thinking, betas, speed, effort, task_budget...） |
| **System Prompt** | 纯字符串 | 多段式，带 cache_control 标记的 ContentBlock 数组 |
| **流式处理** | 无 | 5 种事件类型分别处理，tool_use 的 JSON 增量拼接 |
| **超时/重试** | 无 | 90 秒空闲看门狗 + 自动重试 + fallback 到非流式 |
| **模型回退** | 无 | 主模型过载时自动切换到 fallback 模型 |
| **缓存** | 无 | 请求级 cache breakpoints（prompt caching） |

### 流式的意义

教学版等模型完整回复后才开始执行工具。产品版在模型**还在流式输出时**就已经开始：
1. 解析已完成的 `tool_use` 块
2. 启动工具执行（`StreamingToolExecutor`）
3. 向用户 UI 实时渲染文本

这意味着一个"创建文件 → 验证 → 回复"的三步操作，产品版的延迟体验远优于教学版。

---

## 3. 工具定义：`TOOLS = [{ bash }]` → 40+ 工具体系

### S01 教学版

```python
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

一个工具，一个参数，没有验证，没有权限。

### 产品版：`src/tools/BashTool/BashTool.tsx`（~1000 行，仅 Bash 一个工具）

产品版的 Bash 工具远比教学版复杂：

```typescript
// 工具定义框架（所有工具遵循同一接口）
export const BashTool = buildTool({
  name: BASH_TOOL_NAME,              // "Bash"
  prompt: getSimplePrompt(),         // 动态生成的详细 prompt（含提交指南、安全规则等）
  inputSchema: z.object({            // Zod schema 运行时验证
    command: z.string(),
    timeout: z.number().optional(),
    run_in_background: z.boolean().optional(),
    description: z.string().optional(),
  }),
  validateInput(input) { ... },      // 自定义验证（危险命令、只读模式等）
  async call(input, context) { ... },// 实际执行逻辑
  mapToolResultToToolResultBlockParam(result) { ... },
  isConcurrencySafe(input) { ... },  // 是否可并行
  interruptBehavior() { ... },       // 中断行为
  // ...更多方法
})
```

### 工具体系全景

```
src/tools/
├── BashTool/           — Shell 命令执行（对应 S01 的唯一工具）
├── FileReadTool/       — 读文件（比 cat 更安全、有行号）
├── FileEditTool/       — 编辑文件（精确字符串替换）
├── FileWriteTool/      — 写文件
├── GlobTool/           — 文件名模式匹配
├── GrepTool/           — 内容搜索
├── AgentTool/          — 子智能体
├── TodoWriteTool/      — 任务列表
├── NotebookEditTool/   — Jupyter Notebook 编辑
├── MCPTool/            — MCP 协议工具（动态加载外部工具）
├── SleepTool/          — 定时等待
├── SkillTool/          — 技能系统
├── ... 等 40+ 工具
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **工具数量** | 1 个 | 40+ 个（含 MCP 动态工具） |
| **输入验证** | 无 | Zod schema + 自定义 `validateInput()` |
| **权限系统** | 无 | `canUseTool()` 多层权限：用户交互 / 自动分类器 / hooks |
| **并发控制** | 串行 | `isConcurrencySafe()` 判断，只读工具并行，写工具串行 |
| **中断支持** | 无 | `interruptBehavior()`: cancel / block |
| **工具描述** | 一句话 | 数百字的 prompt，包含使用指南和安全规则 |
| **结果格式** | 纯字符串 | `ToolResultBlockParam`（支持文本、图片、结构化输出） |

---

## 4. 工具执行：`run_bash()` → 分层执行架构

### S01 教学版

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=120)
        return (r.stdout + r.stderr).strip()[:50000] or "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```

一个函数，字符串匹配安全检查，同步执行，截断输出。

### 产品版：三层架构

```
第 1 层：toolOrchestration.ts  — 调度层（决定并行还是串行）
第 2 层：toolExecution.ts      — 执行层（权限检查 + hooks + 执行）
第 3 层：BashTool.call()       — 工具层（实际 shell 执行）
```

#### 第 1 层：调度（`src/services/tools/toolOrchestration.ts`）

```typescript
export async function* runTools(
  toolUseBlocks: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate> {
  // 按并发安全性分批
  const batches = partitionToolCalls(toolUseBlocks, tools)

  for (const batch of batches) {
    if (batch.isConcurrentSafe) {
      // 只读工具并行执行（最多 10 个并发）
      yield* runConcurrently(batch.tools, ...)
    } else {
      // 写操作串行执行
      yield* runSerially(batch.tools, ...)
    }
  }
}
```

#### 第 2 层：执行生命周期（`src/services/tools/toolExecution.ts`）

```
  ┌─ 1. 输入验证（Zod schema）
  ├─ 2. 工具自定义验证（validateInput）
  ├─ 3. Pre-Tool Hooks（执行前钩子）
  ├─ 4. 权限检查（canUseTool → 用户确认 / 自动分类器）
  ├─ 5. 工具执行（tool.call()）
  ├─ 6. 结果映射（mapToolResultToToolResultBlockParam）
  ├─ 7. Post-Tool Hooks（执行后钩子）
  └─ 8. 错误处理 + 遥测
```

#### 第 3 层：BashTool.call()（简化版）

```typescript
async call(input, context, canUseTool, assistantMessage, onProgress) {
  // 1. 安全检查：AST 解析命令（不是字符串匹配！）
  const securityResult = parseForSecurity(command)

  // 2. 沙箱决策
  if (shouldUseSandbox(command)) {
    return SandboxManager.exec(command, ...)
  }

  // 3. 执行
  const result = await exec(command, {
    cwd: context.cwd,
    timeout: timeoutMs,
    onProgress,                    // 实时进度回调
  })

  // 4. 输出处理
  // - 图片检测和调整
  // - 大输出存储到磁盘（超过阈值写文件，返回引用）
  // - Git 操作追踪
  // - 文件修改检测和历史记录
  return processedResult
}
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **安全检查** | 字符串包含匹配（5 个关键字） | AST 解析（解析 shell 语法树检测危险模式） |
| **沙箱** | 无 | 可选沙箱隔离（SandboxManager） |
| **执行方式** | `subprocess.run` 同步阻塞 | 异步执行 + 进度回调 + 后台任务支持 |
| **输出处理** | 截断到 50000 字符 | 大输出存磁盘 + 返回引用路径 + 预览 |
| **并发** | 逐个串行 | 读工具并行（上限 10）、写工具串行 |
| **权限** | 无 | 多层权限：用户确认 / 自动分类 / hooks / 配置规则 |
| **Hooks** | 无 | PreToolUse / PostToolUse / PostToolUseFailure |
| **进度** | 无 | 2 秒后显示实时进度（`onProgress` 回调） |

### 流式工具执行器（`StreamingToolExecutor`）

这是产品版最独特的优化——教学版完全没有的概念：

```
教学版：   模型完整回复 ──────→ 逐个执行工具 ──────→ 下一轮
产品版：   模型流式输出 ──┬──→ 工具 A 已完成，立即执行 ──→ 工具 A 结果就绪
                         ├──→ 模型继续输出...
                         └──→ 工具 B 完成，开始执行 ──→ ...
```

模型还在流式输出时，已完成的 `tool_use` 块就被立即送去执行。这种"流水线"设计大幅降低了端到端延迟。

---

## 5. System Prompt：一行字符串 → 动态组装引擎

### S01 教学版

```python
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."
```

一行 f-string，包含：角色定义 + 工作目录 + 行为指令。

### 产品版：`src/constants/prompts.ts`（~900 行）

```typescript
export function getSystemPrompt(): SystemPrompt {
  const sections: string[] = []

  // ====== 可缓存的静态部分 ======
  sections.push(getSimpleIntroSection())       // 角色定义
  sections.push(getSimpleSystemSection())       // 系统信息
  sections.push(getSimpleDoingTasksSection())   // 任务指南
  sections.push(getActionsSection())            // 行为准则
  sections.push(getUsingYourToolsSection())     // 工具使用指南（按已启用的工具动态生成）
  sections.push(getToneAndStyleSection())       // 语气风格
  sections.push(getOutputEfficiencySection())   // 输出效率

  // ====== 动态部分（每次调用可能不同）======
  sections.push(getSessionSpecificGuidanceSection()) // 会话特定指引
  sections.push(loadMemoryPrompt())                  // 记忆系统
  sections.push(computeSimpleEnvInfo())              // 环境信息（CWD、git status、日期...）
  sections.push(getLanguageSection())                // 语言设置
  sections.push(getMcpInstructionsSection())          // MCP 服务器指令
  sections.push(getScratchpadInstructions())          // 草稿板
  // ... 更多

  return asSystemPrompt(sections.join('\n\n'))
}
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **长度** | ~60 字符 | ~20,000+ token（数万字符） |
| **结构** | 单一字符串 | 多段式，静态段可缓存（prompt caching） |
| **角色定义** | 一句话 | 详细角色 + 安全规则 + 版权规则 + 隐私规则 |
| **工具指南** | "Use bash" | 每个工具有专属使用指南，按已启用工具动态组装 |
| **环境注入** | `os.getcwd()` | CWD + git 状态 + 分支 + 最近提交 + 操作系统 + 模型信息 + 日期 |
| **动态性** | 固定 | 按会话状态、权限模式、启用功能、MCP 服务器动态生成 |

### 教学版 "Act, don't explain" 在产品版中的体现

教学版用一句 `"Act, don't explain"` 让模型从被动回答者变成主动执行者。产品版将这个理念展开为详细的行为规范：

```
# Output efficiency
Keep your text output brief and direct. Lead with the answer or action, not the reasoning.
Skip filler words, preamble, and unnecessary transitions. Do not restate what the user said.
```

本质相同，但产品版更精细地控制了模型的输出风格。

---

## 6. REPL：命令行 while 循环 → React 组件

### S01 教学版

```python
if __name__ == "__main__":
    history = []
    while True:
        query = input("\033[36ms01 >> \033[0m")
        if query.strip().lower() in ("q", "exit", ""):
            break
        history.append({"role": "user", "content": query})
        agent_loop(history)
        # 打印最终回复
        for block in history[-1]["content"]:
            if hasattr(block, "text"):
                print(block.text)
```

外层 REPL 循环 + 内层 agent 循环，通过共享 `history` 列表连接。

### 产品版：`src/screens/REPL.tsx`（~4600 行）

```typescript
// 状态管理
const [messages, setMessages] = useState<Message[]>(initialMessages ?? [])
const messagesRef = useRef(messages)  // 同步引用，避免 React 批处理延迟

// 用户输入处理
const onSubmit = useCallback(async (input: string) => {
  // 1. 斜杠命令检测（/commit, /help, ...）
  // 2. 创建 UserMessage
  // 3. 追加到 messages
  // 4. 调用 onQuery()
}, [...])

// 查询核心
const onQuery = useCallback(async (newMessages: Message[]) => {
  // QueryGuard 防止并发查询
  setMessages(old => [...old, ...newMessages])

  // 调用 queryLoop 并流式处理
  for await (const event of queryLoop(...)) {
    // 实时更新 UI：
    // - 流式文本渲染
    // - 工具调用进度显示
    // - thinking 块展示
  }

  // 完成后清理
  resetLoadingState()
}, [...])
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **技术栈** | Python `input()` | React + Ink（终端 React 渲染器） |
| **状态管理** | 纯列表 | React useState + useRef（双重存储防延迟） |
| **输入处理** | 原始字符串 | 斜杠命令解析 + 多模态输入（文本/图片/文件） |
| **并发控制** | 无（阻塞式） | `QueryGuard` 状态机防止并发查询 |
| **中断** | Ctrl+C 退出 | ESC 优雅中断当前查询，保留历史 |
| **渲染** | `print()` | 流式渲染 + 工具进度 + thinking 块 + Markdown |
| **历史** | 内存列表 | 持久化到磁盘 + 支持 `/resume` 恢复 |

---

## 7. 消息管理：直接追加 → 规范化流水线

### S01 教学版

```python
# 追加 assistant 回复
messages.append({"role": "assistant", "content": response.content})

# 追加工具结果
messages.append({"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": block.id, "content": output}
]})
```

直接构造字典，直接追加。无验证，无转换。

### 产品版：`src/utils/messages.ts`

```typescript
// 内部消息格式 → API 消息格式
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools,
): NormalizedMessage[] {
  // 1. 过滤虚拟消息（isVirtual）
  // 2. 过滤系统消息（保留 local_command 变体）
  // 3. 合并连续的 user 消息（API 要求严格交替）
  // 4. 提升 tool_result 到消息内容开头
  // 5. 将非 tool_result 内容折叠进 tool_result.content
  //    （防止连续 human turn 教模型发 stop sequence）
  // 6. 清理不可用的工具引用
  // 7. 规范化 assistant 内容
  // 8. 处理错误（PDF/图片过大时剥离问题块）
}
```

### 消息生命周期

```
用户输入
  ↓ createUserMessage()
内部 UserMessage（带 uuid, timestamp, isMeta, origin 等元数据）
  ↓ 追加到 messages 数组
queryLoop 读取
  ↓ normalizeMessagesForAPI()
API 格式（严格 user/assistant 交替，tool_result 在 user 消息中）
  ↓ prependUserContext()
最终发送给 API
  ↓
API 返回流式响应
  ↓ normalizeContentFromAPI()
内部 AssistantMessage（解析 tool_use JSON，规范化 content blocks）
  ↓ 追加到 messages 数组
```

### 关键差异

| 维度 | S01 教学版 | 产品版 |
|------|-----------|--------|
| **消息创建** | 手动构造 dict | `createUserMessage()` / `createAssistantAPIErrorMessage()` 工厂函数 |
| **格式转换** | 无（内外同构） | 双向转换：`normalizeMessagesForAPI()` / `normalizeContentFromAPI()` |
| **元数据** | 无 | uuid, timestamp, isMeta, isVirtual, origin, sourceToolAssistantUUID |
| **交替保证** | 隐式（代码结构保证） | 显式合并连续同角色消息 |
| **tool_result 位置** | 直接放 content 数组 | 提升到数组开头 + 内容折叠 |
| **大小控制** | 截断到 50000 字符 | 工具结果预算 + 微压缩 + 磁盘存储大输出 |

---

## 8. 总结：教学版的"不变骨架"在产品版中的体现

S01 学习笔记提出了 5 个设计原则。让我们逐一验证它们在产品版中是否成立：

### 原则 1：模型即决策者

> "何时调用工具、调用什么工具、何时停止，全由模型决定。"

**产品版验证**：✅ 成立，但有边界

模型仍然是主要决策者，但产品版增加了"护栏"：
- **权限系统**：模型想执行 `rm -rf`，但 `canUseTool()` 会阻止
- **Stop Hooks**：模型想停止，但 hook 可以注入错误让它继续
- **Token Budget**：模型想停止，但 budget 未用完时会注入 nudge 继续
- **Max Turns**：超过最大轮数时强制停止

产品版的理念更像："模型决策，harness 把关。"

### 原则 2：循环不变性

> "后续课程都在此循环上叠加机制，但骨架始终不变。"

**产品版验证**：✅ 完全成立

`queryLoop()` 的核心仍然是：
```
while (true) → 调 API → 检查是否需要继续 → 执行工具 → 追加结果 → 继续
```

1500 行中的绝大部分是在这个骨架的"缝隙"中插入功能：
- 调 API **前**：压缩、折叠、缓存
- 调 API **后**：错误恢复、hooks
- 执行工具 **前**：权限检查
- 执行工具 **后**：附件注入、记忆预取
- 追加结果 **后**：任务摘要、MCP 刷新

### 原则 3：累积式上下文

> "messages 只追加不删减。"

**产品版验证**：⚠️ 部分成立

产品版的 messages **会**被压缩/裁剪：
- `autocompact`：将长历史摘要为短文本
- `microcompact`：裁剪工具结果中的冗余
- `contextCollapse`：折叠旧上下文
- `snip`：裁剪超长历史

这正是 S01 笔记中提到的"至少在 s01 中不删减"——后续课程（如 context compression）会改变这一行为。产品版已经实现了这些后续课程的内容。

### 原则 4：工具结果即反馈

> "工具输出以 tool_result 回传给模型，形成闭环。"

**产品版验证**：✅ 完全成立

产品版不仅回传工具结果，还额外注入：
- 文件变更附件（`edited_text_file`）
- 记忆预取结果
- 技能发现结果
- 队列命令通知
- 这些都作为 `user` 消息中的附加内容，丰富了模型的反馈信息

### 原则 5：优雅降级

> "run_bash 永远返回字符串，永远不抛异常。"

**产品版验证**：✅ 完全成立，且更彻底

产品版的每一层都遵循这个原则：
- `toolExecution.ts` 的 catch 块将所有异常转为错误消息
- `queryLoop` 的 catch 块将 API 错误转为 `AssistantAPIErrorMessage`
- 流式中断转为 `UserInterruptionMessage`
- 甚至 413（请求过大）错误也不会崩溃——而是触发压缩恢复

---

## 附：核心文件速查表

| S01 概念 | 产品版文件 | 核心函数/类 | 行数 |
|----------|-----------|------------|------|
| agent_loop() | `src/query.ts` | `queryLoop()` | ~1500 |
| client.messages.create() | `src/services/api/claude.ts` | `queryModelWithStreaming()` | ~2400 |
| TOOLS 定义 | `src/tools/**/*` | 各工具 `buildTool({...})` | 每个工具 200-1000 |
| run_bash() 调度 | `src/services/tools/toolOrchestration.ts` | `runTools()` | ~190 |
| run_bash() 执行 | `src/services/tools/toolExecution.ts` | `runToolUse()` | ~300 |
| run_bash() 流式 | `src/services/tools/StreamingToolExecutor.ts` | `StreamingToolExecutor` | ~400 |
| SYSTEM prompt | `src/constants/prompts.ts` | `getSystemPrompt()` | ~900 |
| REPL 循环 | `src/screens/REPL.tsx` | `onQuery()` / `onSubmit()` | ~4600 |
| messages 管理 | `src/utils/messages.ts` | `normalizeMessagesForAPI()` | ~2400 |
| 依赖注入 | `src/query/deps.ts` | `productionDeps()` | ~41 |

---

*报告生成时间：2026-03-31*
*源码版本：基于当前仓库 main 分支 (4b9d30f)*
