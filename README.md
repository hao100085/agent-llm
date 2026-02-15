# 基于 LLM 的 Agent 系统设计方案

## 1. 目标与设计原则

本方案设计一个可扩展、可观测、可控的 LLM Agent 系统，用于处理“自然语言任务 → 规划 → 工具执行 → 结果交付”的完整闭环。

核心目标：

1. **高可用**：支持超时、重试、降级与熔断。
2. **高准确**：通过任务分解、工具调用校验和反思机制降低幻觉。
3. **高可控**：策略引擎与权限系统约束 Agent 行为。
4. **高扩展**：支持插件化工具、模型路由、多 Agent 协作。
5. **高可观测**：全链路日志、指标、追踪与回放能力。

---

## 2. 总体架构

```text
[Client/API]
    |
    v
[Gateway & Auth] ---> [Policy Engine]
    |
    v
[Orchestrator]
  |     |      |
  |     |      +--> [Memory Service]
  |     +---------> [Planner]
  +---------------> [Model Router]
            |
            v
       [LLM Providers]
            |
            v
      [Tool Runtime] <--> [Tool Registry]
            |
            v
      [Verifier/Critic]
            |
            v
        [Response Builder]
            |
            v
         [Client/API]

(旁路支撑：Observability, Vector DB, Cache, Queue, Secret Manager)
```

---

## 3. 核心模块设计

### 3.1 Gateway & Auth

- 提供 REST / WebSocket / gRPC 接口。
- 负责鉴权（API Key、OAuth2、JWT）与租户隔离。
- 对请求做基本限流和配额控制。

### 3.2 Policy Engine（策略引擎）

- 根据用户角色、任务类型、数据敏感级别决定：
  - 是否允许调用某类工具；
  - 可访问的数据范围；
  - 最大 token / 最大执行步数 / 最大费用。
- 可采用策略即代码（如 OPA/Rego）实现动态策略更新。

### 3.3 Orchestrator（编排器）

- 系统中枢，负责：
  - 状态机驱动（`INIT -> PLAN -> ACT -> VERIFY -> RESPOND`）；
  - 管理重试、回退、超时与中断；
  - 协调 Planner、Tool Runtime、Verifier、Memory。
- 建议采用事件驱动架构，关键事件写入队列以便回放。

### 3.4 Planner（任务规划器）

- 使用 LLM 将目标任务拆分为可执行子任务（Task Graph / DAG）。
- 支持两类规划：
  1. **一次性完整规划**：任务边界清晰时使用；
  2. **滚动规划（RePlan）**：每完成一步重新评估下一步。
- 输出结构化计划（JSON Schema），确保可机读。

### 3.5 Model Router（模型路由）

- 按任务类型、预算、延迟要求选择模型：
  - 复杂推理：高能力模型；
  - 批处理抽取：经济模型；
  - 安全审查：专用分类模型。
- 路由策略可基于规则 + 在线评估（A/B 与 Bandit）。

### 3.6 Tool Registry & Tool Runtime

- **Tool Registry**：保存工具元数据（名称、输入输出 Schema、权限、成本）。
- **Tool Runtime**：负责工具执行与沙箱隔离。
- 工具分层：
  - 内置工具（搜索、计算、代码执行、数据库查询）；
  - 企业工具（工单、CRM、BI、内部知识库）。
- 关键机制：
  - 参数校验（JSON Schema Validation）；
  - 幂等键（避免重复执行）；
  - 审计日志（输入、输出、调用方、耗时）。

### 3.7 Memory Service（记忆系统）

- **短期记忆**：当前会话上下文（窗口内历史）。
- **长期记忆**：用户偏好、任务记录、知识摘要（存于向量库 + KV）。
- 记忆写入策略：
  - 只写入高价值信息（偏好、事实、约束）；
  - 对过期内容打标签并清理。

### 3.8 Verifier/Critic（验证与反思）

- 独立于执行链路，对中间结果做质量检查：
  - 事实一致性；
  - 引用完整性；
  - 工具结果是否满足目标。
- 当失败时触发：
  - 重试当前步骤；
  - 更换工具；
  - 回滚并重新规划。

### 3.9 Observability（可观测性）

- 指标：成功率、平均步数、平均时延、工具错误率、token 成本。
- 日志：结构化日志（trace_id, span_id, user_id, task_id）。
- 链路追踪：OpenTelemetry + 可视化仪表盘。
- 支持任务回放与“决策解释”输出。

---

## 4. 多 Agent 协作模式

适用于复杂任务（例如“市场调研 + 财务分析 + 报告生成”）：

- **Supervisor Agent**：任务总控与分派。
- **Research Agent**：检索与事实收集。
- **Analyst Agent**：数据分析与推理。
- **Writer Agent**：结构化输出与润色。
- **Critic Agent**：质量审查。

通信方式：

1. 共享黑板（Blackboard）存放中间结论；
2. 消息总线进行异步协作；
3. Supervisor 根据质量门限决定是否进入下一阶段。

---

## 5. 执行流程（单 Agent）

1. 用户提交目标与约束（预算、时限、格式）。
2. Gateway 完成鉴权并注入租户上下文。
3. Policy Engine 下发执行边界。
4. Planner 生成任务分解（结构化 DAG）。
5. Orchestrator 逐步执行：
   - 调用模型生成行动；
   - 调用工具并写入结果；
   - 触发 Verifier 质量检查。
6. 若检查失败，进入 RePlan / Retry。
7. 通过检查后生成最终答复，并附带来源与执行摘要。
8. 高价值内容写入长期记忆。

---

## 6. 数据模型（示例）

### 6.1 Task

```json
{
  "task_id": "t_123",
  "user_id": "u_001",
  "goal": "整理竞品分析报告",
  "constraints": {
    "max_budget": 2.5,
    "deadline_sec": 120,
    "output_format": "markdown"
  },
  "status": "running"
}
```

### 6.2 Plan Step

```json
{
  "step_id": "s_02",
  "type": "tool_call",
  "tool": "web_search",
  "input": {
    "query": "2025 AI Agent 市场规模"
  },
  "depends_on": ["s_01"],
  "status": "pending"
}
```

---

## 7. 安全与风控设计

1. **Prompt 注入防护**：
   - 系统提示词与用户输入分层；
   - 对外部网页内容进行“非可信标记”；
   - 禁止外部内容覆盖系统指令。
2. **工具权限最小化**：
   - 默认拒绝策略；
   - 按角色 + 场景开放工具。
3. **敏感信息防泄漏**：
   - 输出前 DLP 检测（PII、密钥、合同字段）；
   - 日志脱敏。
4. **执行沙箱隔离**：
   - 代码执行工具运行在隔离容器；
   - 资源限制（CPU/内存/网络白名单）。

---

## 8. 评估体系

- **离线评测**：
  - 任务成功率（Task Success）；
  - 工具调用正确率（Tool Accuracy）；
  - 事实一致性（Faithfulness）。
- **在线评测**：
  - 用户满意度、任务完成时长、复用率。
- **回归机制**：
  - 每次 Prompt/策略变更都跑基准集。

---

## 9. 技术栈建议

- 服务层：Python (FastAPI) 或 Node.js (NestJS)
- 编排层：Temporal / LangGraph / 自研状态机
- 向量库：pgvector / Milvus / Weaviate
- 缓存：Redis
- 消息队列：Kafka / RabbitMQ
- 可观测：OpenTelemetry + Prometheus + Grafana
- 部署：Kubernetes + Helm

---

## 10. 最小可行版本（MVP）路线图

### Phase 1（2~4 周）

- 单 Agent + 3 个基础工具（搜索、网页抓取、计算）
- 会话记忆 + 简单长期记忆
- 基本可观测（日志 + 成本统计）

### Phase 2（4~8 周）

- 引入 Verifier 与 RePlan
- 工具权限系统与审计看板
- 模型路由（按成本/延迟）

### Phase 3（8~12 周）

- 多 Agent 协作
- 更完整安全体系（注入检测、DLP）
- A/B 实验平台与自动回归

---

## 11. 关键实现建议（实践版）

1. **先做可观测，再追求复杂智能**：没有 trace 的 Agent 难以优化。
2. **所有工具都必须 Schema 化**：把非结构化错误扼杀在接口层。
3. **先约束后放权**：严格策略边界比“自由调用”更稳定。
4. **把 RePlan 作为默认能力**：真实环境中的计划总会偏离。
5. **建立失败样本库**：持续喂给 Verifier 和路由器，提升迭代效率。

该方案可从 MVP 快速起步，并平滑演进到企业级多 Agent 平台。
