# ACCK 架构

## 目的

用图看清进程里有哪些部件，以及提交、冲突、效应、重放如何衔接。

## 读者

读实现之前需要先看结构的人。

## 范围

图表达的是 `docs/FEATURES.md` 与 `docs/DATABASE.md` 里已经定下的路径。不另画 Kubernetes。

## 内容

HTTP、MCP、LangGraph 和协作层进入同一个进程。注册表在内存里。订单只有 `Kernel.commit` 能按 11 步写入。协作层调用模型，再调用提交或工具。`no_kernel` 不经过 `Kernel.commit`。

```mermaid
flowchart LR
  HTTP[HTTP /v1] --> COMMIT[Kernel.commit]
  HTTP --> INVOKE[invoke_tool]
  MCP["POST /mcp"] --> INVOKE
  LG[LangGraph] --> COLLAB[协作层]
  COLLAB --> LLM[LlmClient]
  COLLAB --> COMMIT
  COLLAB --> INVOKE
  COMMIT --> REG[Registry]
  INVOKE --> REG
  COMMIT --> PG[(PostgreSQL)]
  INVOKE --> PG
```

提交时先处理结构、任务状态和幂等，再锁住订单行。预算和效应引用在版本策略之前。文档约束在应用 patch 之后。

```mermaid
sequenceDiagram
  participant C as 调用方
  participant K as Kernel.commit
  participant DB as documents 行锁
  C->>K: PatchIntent
  alt 任务不存在
    K-->>C: 404
  else 幂等命中
    K-->>C: 第一次的结果
  else 继续
    K->>DB: SELECT FOR UPDATE
    K->>K: 路径、预算、效应引用、版本策略
    alt 通过
      K->>DB: version+1，账本，检查点
    else 拒绝
      K->>DB: 订单不变，仍写账本
    end
  end
```

```mermaid
stateDiagram-v2
  [*] --> Compare
  Compare --> Apply: base_version 等于当前版本
  Compare --> Policy: 落后或大于当前版本
  Policy --> RejectRunning: reject
  Policy --> Replan: replan 且未达上限
  Policy --> Aborted: replan 且达到上限
  Policy --> Disjoint: rebase
  Disjoint --> Apply: 路径不相交
  Disjoint --> RejectRunning: 路径相交
  Apply --> Committed: patch 与文档约束通过
```

写类工具才进入效应状态。`READ` 只写轨迹。

```mermaid
stateDiagram-v2
  [*] --> executed: 真实调用
  executed --> committed: 成功提交引用
  executed --> compensated: 补偿成功
  note right of executed
    补偿失败则停在 executed
  end note
```

`LIVE` 写轨迹。`STRICT` 只比较 `input_hash`，第一处不等即停，不改订单。`RESUME` 从最大检查点继续，并删掉该游标之后的轨迹。检查顺序的理由在 `docs/ADR/002-gate-order.md`。
