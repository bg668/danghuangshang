# 🔗 多 Agent 通信机制详解

> ← [返回架构详解](./architecture.md) | [任务状态机 →](./task-state-machine.md)

本文系统性梳理「当皇上」项目中多 Agent 协作的**通信方式**、**上下文维护机制**和**任务进度追踪**实现原理，结合 JavaScript/TypeScript/Python 实现给出关键代码片段。

---

## 目录

1. [整体通信架构](#1-整体通信架构)
2. [Agent 间通信方式](#2-agent-间通信方式)
   - [2.1 OpenClaw 框架原生通信](#21-openclaw-框架原生通信-sessions_spawn--sessions_send)
   - [2.2 消息总线（发布/订阅）](#22-消息总线发布订阅-message-bus)
   - [2.3 Discord/飞书 @mention 通信](#23-discordfeishu-mention-通信)
3. [上下文维护机制](#3-上下文维护机制)
   - [3.1 身份注入（Identity Injection）](#31-身份注入identity-injection)
   - [3.2 工作区与持久化记忆](#32-工作区与持久化记忆)
   - [3.3 上下文压缩（Context Compressor）](#33-上下文压缩context-compressor)
4. [任务进度追踪](#4-任务进度追踪)
   - [4.1 Task Store 状态机](#41-task-store-状态机)
   - [4.2 依赖管理与上游输出聚合](#42-依赖管理与上游输出聚合)
   - [4.3 错误处理与重试](#43-错误处理与重试)
5. [权限与访问控制](#5-权限与访问控制)
6. [完整协作流程示例](#6-完整协作流程示例)
7. [通信机制对比总结](#7-通信机制对比总结)

---

## 1. 整体通信架构

本项目以**明朝三省六部制**为蓝本，构建了多层次的 Agent 通信体系：

```
┌─────────────────────────────────────────────────────┐
│              皇帝（用户）                             │
│        Discord / 飞书 / Web UI @mention              │
└──────────────────┬──────────────────────────────────┘
                   │  圣旨（用户指令）
                   ▼
┌─────────────────────────────────────────────────────┐
│          OpenClaw Gateway（门下省/中书省）             │
│  - 消息路由（bindings: channel + accountId → agentId）│
│  - 会话隔离管理                                       │
│  - Cron 调度 / 心跳保活                               │
└──┬────────┬────────┬────────┬────────┬──────────────┘
   │        │        │        │        │
   ▼        ▼        ▼        ▼        ▼
司礼监    兵部     户部     礼部     工部 / 吏部 / 刑部
(调度)   (编码)   (财务)   (营销)   (运维 / 管理 / 法务)
   │
   ├─ sessions_spawn / sessions_send ──→ 内阁 / 都察院
   │
   └─ 消息总线 (MessageBus pub/sub) ──→ 系统内部事件
              ↑↓
        Task Store (JSON 持久化)
        上下文压缩 (Context Compressor)
        权限拦截 (Permission Guard / 门下省)
```

通信分为三层：
| 层次 | 机制 | 适用场景 |
|------|------|---------|
| **应用层** | OpenClaw `sessions_spawn` / `sessions_send` | 跨 Agent 任务派发与结果传递 |
| **事件层** | MessageBus（Node.js EventEmitter） | 系统内部异步事件通知 |
| **交互层** | Discord / 飞书 @mention + Bot 回复 | 用户与 Agent、Agent 之间的可见消息 |

---

## 2. Agent 间通信方式

### 2.1 OpenClaw 框架原生通信：`sessions_spawn` / `sessions_send`

这是本系统**核心的 Agent 间通信方式**，基于 OpenClaw 框架提供的两个原语：

| 原语 | 作用 | 适用场景 |
|------|------|---------|
| `sessions_spawn` | 启动一个新的子 Agent 会话，异步执行任务后返回结果 | 独立子任务、并行派发 |
| `sessions_send` | 向已有会话发送消息（可等待回复） | 顾问角色、需要来回沟通的任务 |

**典型调用模式（司礼监 prompt 片段）：**

```
# 1. 司礼监 → 内阁（sessions_send，顾问模式）
使用 sessions_send 通知内阁：
  请分析以下任务并生成执行 Plan：
  「实现用户登录 API，包含 JWT 鉴权和限流」

# 2. 内阁返回 Plan（步骤列表 + 依赖关系）
{
  "description": "实现用户登录 API",
  "steps": [
    { "id": 1, "agent": "bingbu",   "task": "实现 JWT 登录接口", "dependencies": [] },
    { "id": 2, "agent": "bingbu",   "task": "添加 rate limiting", "dependencies": [1] },
    { "id": 3, "agent": "duchayuan","task": "代码安全审查",        "dependencies": [1,2] }
  ]
}

# 3. 司礼监 → 兵部（sessions_spawn，独立执行）
使用 sessions_spawn 创建兵部会话：
  任务：实现 JWT 登录接口
  要求：RESTful，POST /auth/login，返回 {token, expires}
```

**配置示例**（`openclaw.example.json`）：

```json
{
  "agents": {
    "list": [
      {
        "id": "silijian",
        "subagents": {
          "allowAgents": ["neige", "bingbu", "hubu", "libu", "gongbu", "libu2", "xingbu", "duchayuan"],
          "maxConcurrent": 4
        }
      },
      {
        "id": "bingbu",
        "subagents": {
          "allowAgents": [],
          "maxConcurrent": 0
        }
      }
    ]
  }
}
```

> 关键设计：六部（兵部、户部等）`allowAgents` 为空，**只有司礼监**可以向其他 Agent 派活，防止越级指挥。

---

### 2.2 消息总线（发布/订阅）：Message Bus

文件：`scripts/message-bus.js`

基于 Node.js 原生 `EventEmitter` 实现的**进程内发布/订阅总线**，用于系统组件间的异步事件通知。

**核心实现：**

```javascript
// scripts/message-bus.js
const EventEmitter = require('events');

class MessageBus extends EventEmitter {
  constructor() {
    super();
    this.subscriptions = new Map();   // topic → Set<handler>
    this.messageHistory = [];          // 最近 100 条消息
    this.maxHistory = 100;
  }

  // 订阅主题，返回取消订阅函数
  subscribe(topic, handler) {
    if (!this.subscriptions.has(topic)) {
      this.subscriptions.set(topic, new Set());
    }
    this.subscriptions.get(topic).add(handler);
    this.on(topic, handler);
    return () => this.unsubscribe(topic, handler);
  }

  // 发布消息（附加时间戳和唯一 ID）
  publish(topic, data) {
    const message = {
      topic,
      data,
      timestamp: new Date().toISOString(),
      id: `msg_${Date.now()}_${Math.random().toString(36).substring(2, 11)}`
    };
    this._addToHistory(message);
    this.emit(topic, data, message);
  }

  // 请求-响应模式（带超时）
  async request(topic, data, timeout = 5000) {
    const replyTopic = `reply.${topic}.${Date.now()}`;
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.unsubscribe(replyTopic, handler);
        reject(new Error(`Request timeout: ${topic}`));
      }, timeout);
      const handler = (response) => {
        clearTimeout(timer);
        resolve(response);
      };
      this.subscribe(replyTopic, handler);
      this.publish(topic, { ...data, replyTo: replyTopic });
    });
  }
}

// 全局单例
const messageBus = new MessageBus();
module.exports = { MessageBus, messageBus };
```

**主要事件主题：**

| Topic | 触发时机 | 订阅方 |
|-------|---------|--------|
| `task.created` | 新任务创建后 | 日志、监控 |
| `task.updated` | 步骤状态变更 | 进度通知 |
| `agent.dispatch` | Agent 被派发任务 | 权限审查 |
| `permission.denied` | 权限拦截触发 | 告警 |
| `broadcast` | 全系统广播 | 所有订阅方 |

**使用示例：**

```javascript
const { messageBus } = require('./scripts/message-bus');

// 订阅任务创建事件
const unsub = messageBus.subscribe('task.created', (data) => {
  console.log(`新任务: ${data.taskId}`);
});

// 发布事件
messageBus.publish('task.created', { taskId: 'task_123', plan: {...} });

// 请求-响应（等待某 Agent 回复）
const result = await messageBus.request('agent.neige.optimize', { task: '...' }, 10000);

// 取消订阅
unsub();
```

---

### 2.3 Discord/飞书 @mention 通信

Agent 以 Bot 身份运行在 Discord 频道或飞书群组中，通过 **@mention** 触发指定 Agent：

```
# 用户发起任务
皇帝：@司礼监 帮我分析竞品并输出报告

# 司礼监派发给礼部
司礼监：@礼部 请调研竞品 A、B、C，输出 SWOT 分析报告，明天早上汇报

# 礼部完成后汇报
礼部：@司礼监 已完成竞品分析，报告已存入 Notion，链接：[...]

# 涉及代码的任务
司礼监：@兵部 请实现用户注册 API，PR 提交后 @都察院 自动审查
```

**关键配置**（防止 Bot 互相触发死循环）：

```json
{
  "discord": {
    "allowBots": "mentions"
  }
}
```

- `"allowBots": "mentions"`：Bot 只在被 @ 时响应其他 Bot（默认安全配置）
- `"allowBots": true`：⚠️ **禁止使用**，会导致 Bot 无限互发消息

---

## 3. 上下文维护机制

### 3.1 身份注入（Identity Injection）

每个 Agent 启动时，OpenClaw Gateway 自动将以下内容组装为**系统提示**注入会话：

```
[系统提示组装顺序]
1. SOUL.md       — Agent 的核心价值观、协作原则（全局共享）
2. IDENTITY.md   — 角色身份描述、职责边界
3. 工作区文件    — ~/.clawd-{agentId}/ 目录下的参考文档、历史记录
4. hooks 结果    — handler.ts 动态注入内容（如自我改进提醒）
```

**自我改进 Hook**（`skills/self-improving-agent/hooks/openclaw/handler.ts`）：

```typescript
// 每次会话启动时追加自我改进提醒
export async function handler(context: HookContext): Promise<HookResult> {
  return {
    systemPromptAppend: `
## 自我改进提醒
你上次运行后记录了以下改进点，请在本次任务中注意：
${await loadSelfImprovementNotes(context.agentId)}
    `
  };
}
```

**Agent 身份示例**（`configs/ming-neige/agents/silijian.md`）：

```markdown
## 身份
你是司礼监大内总管，皇帝的首席助理。

## 核心职责
- 接收皇帝圣旨，理解意图
- 召集内阁优化任务描述，生成执行 Plan
- 按 Plan 调度六部执行，追踪进度
- 汇总结果向皇帝汇报

## 协作原则
- 重要任务必须先经内阁审议
- 代码类任务完成后必须经都察院审查
- 任务状态随时更新到 Task Store
```

---

### 3.2 工作区与持久化记忆

每个 Agent 有**独立的工作区目录**，跨会话持久化记忆：

```
~/.clawd-silijian/    # 司礼监工作区
├── memory/           # 持久化记忆文件
│   ├── projects.md   # 正在进行的项目列表
│   ├── preferences.md# 皇帝偏好记录
│   └── history.md    # 重要决策历史
├── tasks/            # 任务相关文件
└── notes.md          # 工作笔记

~/.clawd-bingbu/      # 兵部工作区
├── memory/
│   ├── codebase.md   # 代码库知识
│   └── patterns.md   # 常用代码模式
└── worktrees/        # Git worktree（并行开发）
```

**记忆写入约定（Agent prompt 中约定）：**

```markdown
## 记忆管理规则
- 每次任务完成后，将关键决策写入 memory/history.md
- 发现用户偏好时，更新 memory/preferences.md
- 定期整理，保持文件精简（< 500 行）
```

---

### 3.3 上下文压缩（Context Compressor）

文件：`scripts/context-compressor.js`

解决长任务链**上下文爆炸**问题，基于重要性评分保留关键信息：

**重要性评分权重：**

```javascript
const Config = {
  weights: {
    decision:      10,   // 关键决策（最高优先级）
    artifact:       9,   // 交付物（代码、文档链接）
    result:         9,   // 最终结果
    error:          8,   // 错误信息
    approval:       8,   // 用户确认
    plan:           7,   // 执行计划
    clarification:  3,   // 澄清问答
    brainstorm:     2,   // 头脑风暴
    discussion:     2,   // 讨论过程
    attempt:        1    // 尝试过程（最低优先级）
  },
  thresholds: {
    messageCount: 20,    // 超过 20 条触发压缩
    tokenCount:   4000,  // 超过 4000 token 触发压缩
    ageMinutes:   30     // 超过 30 分钟触发压缩
  },
  keepRatio: 0.4         // 保留 40% 重要消息
};
```

**压缩算法：**

```javascript
function compress(messages, options = {}) {
  // 1. 对每条消息计算重要性评分
  const scored = messages.map((msg, index) => ({
    ...msg,
    index,
    score: calculateImportance(msg)  // 类型权重 × 时间衰减 × 长度因子 × 角色因子
  }));

  // 2. 按重要性排序，保留前 40%
  const keepCount = Math.max(5, Math.floor(messages.length * Config.keepRatio));
  const kept = [...scored].sort((a, b) => b.score - a.score).slice(0, keepCount);

  // 3. 恢复原始时间顺序
  const compressed = kept.sort((a, b) => a.index - b.index);

  // 4. 生成摘要（可接入 LLM）
  const summary = generateSummary(compressed, options);

  return { messages: compressed, summary, stats: { original: messages.length, compressed: compressed.length } };
}

function calculateImportance(message) {
  const type = identifyMessageType(message);
  const baseScore = Config.weights[type] || 5;

  // 时间衰减：1 小时内线性衰减到 0.5
  const age = Date.now() - new Date(message.timestamp || Date.now()).getTime();
  const timeFactor = Math.max(0.5, 1 - (age / (1000 * 60 * 60)));

  // 长度因子：极短消息评分减半
  const lengthFactor = (message.content || '').length > 10 ? 1 : 0.5;

  // 角色因子：用户消息权重提升 20%
  const roleFactor = message.role === 'user' ? 1.2 : 1;

  return baseScore * timeFactor * lengthFactor * roleFactor;
}
```

**压缩效果示例：**

```
原始对话（50 条）：
  兵部：考虑用 JWT 还是 Session...
  兵部：那用 JWT 吧...
  兵部：等等，试试另一种方案...
  ...（45 条讨论/尝试）...
  兵部：✅ 完成！代码链接：[...]

压缩后（5 条 + 摘要）：
  [摘要] 技术选型：JWT；尝试方案：3种；遇到问题：rate limiting（已解决）
  兵部：✅ 完成！代码链接：[...]

压缩率：90%（50条 → 5条 + 摘要）
```

**使用命令：**

```bash
node scripts/context-compressor.js compress \
  --input bingbu_conversation.json \
  --output compressed.json
```

---

## 4. 任务进度追踪

### 4.1 Task Store 状态机

文件：`scripts/task-store.js`

基于 JSON 文件的**任务状态持久化存储**，解决多 Agent 协作中的信息孤岛问题。

**任务状态枚举：**

```javascript
const TaskState = {
  PENDING:           'pending',           // 等待执行
  RUNNING:           'running',           // 执行中
  SUCCESS:           'success',           // 成功完成
  FAILED:            'failed',            // 失败（重试已耗尽）
  RETRYING:          'retrying',          // 重试中
  CANCELLED:         'cancelled',         // 已取消
  REVISION_REQUIRED: 'revision_required'  // 需要修改（审查驳回）
};

const ErrorType = {
  TRANSIENT:  'transient',  // 临时错误（网络、限流）→ 自动重试
  PERMANENT:  'permanent',  // 永久错误（Bug、逻辑问题）→ 打回修改
  REJECTED:   'rejected'    // 审查驳回 → revision_required
};
```

**任务数据结构：**

```json
{
  "id": "task_20260321_120000_login_api",
  "status": "running",
  "plan": {
    "description": "实现用户登录 API",
    "steps": [...]
  },
  "steps": [
    {
      "id": 1,
      "agent": "bingbu",
      "task": "实现 JWT 登录接口",
      "dependencies": [],
      "status": "success",
      "output": {
        "result": "登录 API 已实现",
        "codeLink": "https://github.com/xxx/pr/42",
        "artifacts": ["screenshot.png"]
      },
      "createdAt": "2026-03-21T12:00:00Z",
      "completedAt": "2026-03-21T12:02:30Z",
      "retryCount": 0,
      "error": null
    },
    {
      "id": 2,
      "agent": "duchayuan",
      "task": "代码安全审查",
      "dependencies": [1],
      "status": "pending",
      "output": null,
      "retryCount": 0,
      "revisionReason": null
    }
  ],
  "createdAt": "2026-03-21T12:00:00Z",
  "updatedAt": "2026-03-21T12:02:30Z",
  "completedAt": null
}
```

**存储位置：** `~/.clawd/task-store/tasks.json`

---

### 4.2 依赖管理与上游输出聚合

Task Store 的核心特性：**自动聚合上游步骤的输出**，让每个 Agent 知道前置步骤做了什么。

```javascript
// scripts/task-store.js
function getInput(taskId, stepId) {
  const task = readDB().tasks[taskId];
  const stepIndex = task.steps.findIndex(s => s.id === stepId || s.agent === stepId);
  const step = task.steps[stepIndex];

  // 获取依赖步骤（显式声明或默认取前置所有步骤）
  const dependencies = step.dependencies.length > 0
    ? step.dependencies
    : task.steps.slice(0, stepIndex).map(s => s.id);

  // 自动聚合上游输出
  const upstreamOutputs = {};
  for (const depId of dependencies) {
    const depStep = task.steps.find(s => s.id === depId);
    if (depStep?.status !== TaskState.SUCCESS) {
      throw new Error(`依赖步骤 ${depId} 未完成（状态：${depStep?.status}）`);
    }
    upstreamOutputs[depId] = {
      agent: depStep.agent,
      task: depStep.task,
      output: depStep.output,         // 完整输出（代码链接、截图等）
      completedAt: depStep.completedAt
    };
  }

  return {
    taskId,
    stepId: step.id,
    agent: step.agent,
    task: step.task,
    upstreamOutputs,                  // 所有前置步骤的输出
    context: {
      originalTask: task.plan.description,
      totalSteps: task.steps.length,
      currentStep: stepIndex + 1
    }
  };
}
```

**聚合输出示例（步骤 2 调用 `get-input` 的返回值）：**

```json
{
  "taskId": "task_20260321_120000",
  "stepId": 2,
  "agent": "duchayuan",
  "task": "代码安全审查",
  "upstreamOutputs": {
    "1": {
      "agent": "bingbu",
      "task": "实现 JWT 登录接口",
      "output": {
        "result": "登录 API 已实现",
        "codeLink": "https://github.com/xxx/pr/42"
      }
    }
  },
  "context": {
    "originalTask": "实现用户登录 API",
    "totalSteps": 2,
    "currentStep": 2
  }
}
```

**CLI 操作示例：**

```bash
# 创建任务
node scripts/task-store.js create --id task_123 --plan plan.json

# 更新步骤状态（兵部完成后）
node scripts/task-store.js update \
  --task task_123 --step 1 \
  --status success --output output.json

# 获取下一步输入（都察院调用，自动拿到兵部输出）
node scripts/task-store.js get-input --task task_123 --step 2

# 查询任务进度
node scripts/task-store.js status --task task_123

# 列出所有任务
node scripts/task-store.js list --limit 10
```

---

### 4.3 错误处理与重试

```javascript
// 任务整体状态计算逻辑
function calculateTaskStatus(steps) {
  const allSuccess = steps.every(s => s.status === TaskState.SUCCESS);
  const hasFatal  = steps.some(s => s.status === TaskState.FAILED && s.retryCount >= 3);
  const hasRunning = steps.some(s => s.status === TaskState.RUNNING);

  if (allSuccess)  return TaskState.SUCCESS;
  if (hasFatal)    return TaskState.FAILED;
  if (hasRunning)  return TaskState.RUNNING;
  return TaskState.PENDING;
}
```

**错误处理流程：**

```
临时错误（TRANSIENT）：
  兵部 → 网络超时
  → status=retrying, retryCount++
  → 司礼监检测到，自动重派（最多 3 次）
  → 3 次后仍失败 → status=failed, errorType=transient

永久错误（PERMANENT）：
  兵部 → 逻辑 Bug 无法修复
  → status=failed, errorType=permanent
  → 司礼监汇报皇帝，请示处理

审查驳回（REJECTED）：
  都察院 → 代码缺少 rate limiting
  → status=revision_required, revisionReason="缺少 rate limiting"
  → 司礼监打回兵部重做
  → 兵部修改完成 → status=success
  → 都察院重新审查
```

---

## 5. 权限与访问控制

文件：`scripts/permission-guard.js`

基于**明朝内阁制官僚层级**的权限矩阵，扮演"门下省"角色，在任何跨 Agent 调用前进行拦截审查。

**权限矩阵（精简）：**

```javascript
const DEFAULT_PERMISSIONS = {
  // 司礼监：全权限
  silijian: {
    agents:   ['*'],                                    // 可调用所有 Agent
    skills:   ['*'],                                    // 可执行所有 Skill
    taskOps:  ['create', 'update', 'cancel', 'read']
  },

  // 内阁：顾问角色，不直接指挥六部
  neige: {
    agents:   [],                                       // 不直接调用 Agent
    skills:   ['self-improving-agent', 'quadrants'],
    taskOps:  ['read']
  },

  // 兵部：只能用编码相关工具
  bingbu: {
    agents:   [],
    skills:   ['github', 'browser-use', 'self-improving-agent'],
    taskOps:  ['update', 'read']
  },

  // 都察院：独立监察，只读 + 审查报告写入
  duchayuan: {
    agents:   [],
    skills:   ['github', 'self-improving-agent'],
    taskOps:  ['update', 'read'],
    fileAccess: { read: ['*'], write: ['~/.clawd-duchayuan/', 'reviews/'] }
  }
};
```

**权限拦截示例：**

```javascript
const { permissionGuard } = require('./scripts/permission-guard');

// ✅ 准奏：司礼监可以调用兵部
permissionGuard.checkAgentCall('silijian', 'bingbu');

// ❌ 驳回：户部不能直接调用兵部（必须经司礼监）
permissionGuard.checkAgentCall('hubu', 'bingbu');
// → throws AppError(PERMISSION_DENIED): "[门下省] 驳回：hubu 无权调用 bingbu"

// ✅ 准奏：兵部可以使用 github skill
permissionGuard.checkSkillExecution('bingbu', 'github');

// ❌ 驳回：礼部不能使用 github skill
permissionGuard.checkSkillExecution('libu', 'github');
// → throws AppError(PERMISSION_DENIED)
```

**权限拒绝后通过 MessageBus 广播事件：**

```javascript
// permission-guard.js 内部
messageBus.publish('permission.denied', {
  caller: 'hubu',
  target: 'bingbu',
  action: 'agent_call',
  timestamp: new Date().toISOString()
});
```

**CLI 用法：**

```bash
# 检查权限
node scripts/permission-guard.js check \
  --agent silijian --action agent_call --target bingbu

# 查看某 Agent 权限摘要
node scripts/permission-guard.js summary --agent bingbu

# 列出所有权限配置
node scripts/permission-guard.js list

# 查看审计日志
node scripts/permission-guard.js audit --limit 20
```

---

## 6. 完整协作流程示例

以「实现用户登录 API 并审查」为例，展示完整的通信链路：

```
1. 用户指令
   Discord: @司礼监 实现用户登录 API，含 JWT 鉴权

2. 司礼监 → 内阁（sessions_send）
   "请分析任务，生成执行 Plan"
   内阁返回：
   {
     steps: [
       {id:1, agent:"bingbu",    task:"实现 JWT 登录接口", dependencies:[]},
       {id:2, agent:"duchayuan", task:"代码安全审查",       dependencies:[1]}
     ]
   }

3. 司礼监创建 Task Store 记录
   node task-store.js create --id task_20260321 --plan plan.json

4. 权限检查（Permission Guard）
   permissionGuard.checkAgentCall('silijian', 'bingbu')   // ✅ 准奏
   permissionGuard.checkAgentCall('silijian', 'duchayuan') // ✅ 准奏

5. 司礼监 → 兵部（sessions_spawn）
   "任务：实现 JWT 登录接口
    参考：task_20260321 步骤 1"

6. 兵部执行（独立会话）
   - 写代码，提交 PR
   - 完成后调用：
     node task-store.js update \
       --task task_20260321 --step 1 \
       --status success --output bingbu_out.json

7. MessageBus 事件广播
   messageBus.publish('task.updated', {taskId: 'task_20260321', step: 1, status: 'success'})

8. 司礼监 → 都察院（sessions_spawn，步骤 1 完成后）
   node task-store.js get-input --task task_20260321 --step 2
   → 自动拿到兵部输出（PR 链接、代码）
   "任务：代码安全审查
    待审代码：[兵部 PR 链接]
    重点检查：JWT 密钥管理、SQL 注入、rate limiting"

9. 都察院审查
   a. 通过 → node task-store.js update --step 2 --status success
   b. 驳回 → node task-store.js update --step 2 --status revision_required
               --revision-reason "缺少 rate limiting"
            → 司礼监 @兵部 "都察院驳回，需补充 rate limiting"
            → 兵部修改 → 重新审查

10. 任务完成，司礼监汇报
    node task-store.js status --task task_20260321
    Discord: @皇帝 登录 API 已完成并通过代码审查，PR：[链接]
```

---

## 7. 通信机制对比总结

| 机制 | 类型 | 方向 | 持久化 | 适用场景 |
|------|------|------|--------|---------|
| `sessions_spawn` | 同步/异步任务派发 | 单向（调用方 → 子 Agent） | 否（会话内） | 独立子任务，需要完整上下文隔离 |
| `sessions_send` | 消息发送（可等待回复） | 双向（会话间通信） | 否（会话内） | 顾问模式，来回沟通的任务 |
| MessageBus（pub/sub） | 异步事件 | 广播/点对点 | 有限（最近 100 条） | 系统内部事件通知，解耦组件 |
| Discord/飞书 @mention | 人机 + Agent 间消息 | 双向，公开可见 | 平台存储 | 用户交互，跨 Agent 可见协作 |
| Task Store（JSON） | 状态共享存储 | 全局读写 | 是（`~/.clawd/task-store/`） | 任务进度、依赖输出传递 |
| Permission Guard（门下省） | 访问控制拦截 | 同步拦截 | 审计日志 | 跨 Agent 调用权限验证 |
| Context Compressor | 上下文优化 | 预处理 | 否（按需调用） | 长任务链上下文压缩 |

**设计理念总结：**

1. **职责分离**：司礼监负责调度，六部只管执行，都察院独立审查——信息流单向，防止循环依赖。
2. **状态外置**：任务状态存储在 Task Store，而非 Agent 内部——任何 Agent 随时可查询全局进度。
3. **显式依赖**：上游输出通过 `get-input` 自动聚合——Agent 无需手动传递，杜绝遗漏。
4. **上下文优化**：长对话自动压缩，保留决策/交付物，丢弃讨论/尝试——降低 token 消耗。
5. **权限层级**：基于官制的权限矩阵，防止越级调用——可审计，可追溯。

---

## 🔗 相关文档

- [架构详解](./architecture.md) — 三省六部映射与系统总览
- [任务状态机](./task-state-machine.md) — Task Store 完整使用指南
- [语义记忆搜索](./memory-search.md) — Agent 长期记忆检索
- [Discord 安全配置](./discord-safety.md) — Bot 通信安全设置

← [返回架构详解](./architecture.md) | [任务状态机 →](./task-state-machine.md)
