# ADR 0005：插件只替换门后面的实现

## 目的

说明五类插件能改什么，以及为什么不能改检查顺序或直接写订单。

## 读者

要加工具、冲突策略、文档约束、模型或编排适配器的人。

## 范围

只覆盖 `docs/FEATURES.md` 第 1 节和 `docs/DEVELOPMENT.md` 第 2 节的五个 Protocol。

## 内容

### 状态

已接受。

### 背景

订单字段、冲突时的动作、工具副作用和模型输出都会变。若这些判断写进 `commit.py`，每换一个文档域就要改提交事务，测试无法用一个与订单无关的假工具证明内核没被改。

若插件可以重排门，或可以自己 `UPDATE documents`，ADR 002 的顺序和「唯一写入口」就不再成立。超预算的提交也可以先被某个插件合并进订单。

### 决定

进程内注册表只保存五类对象：

| 插件 | 替换的是 |
|------|----------|
| `ToolPlugin` | 工具被允许执行之后的 `invoke` |
| `ConflictPolicyPlugin` | 版本落后时返回 `reject`、`replan` 或 `rebase`。不改数据库 |
| `DocumentSpec` | 初值、角色路径、patch 应用之后的文档约束 |
| `LlmClient` | 模型补全 |
| `OrchestratorAdapter` | 外部图如何调用提交和工具 |

同名第二次注册失败，不覆盖。HTTP、进程内调用和 `POST /mcp` 用同一份表，并且在同一个 API 进程里。

`commit.py` 不出现订单字段判断。路径和状态迁移放在 `order` 这份 `DocumentSpec` 里。策略函数只返回动作，版本加一和写账本仍由内核做。

### 后果

本版内置名字是 `read_order`、`place_order`、`refund`、`send_receipt`、`abort`、`refresh_and_replan`、`merge_if_disjoint`、`order`、LangGraph，以及脚本模型或 `ACCK_LLM=live`。

没有卸载、热更新，也没有按任务分开的注册表。

未注册不另立异常类。创建任务时名字未注册，HTTP 422，响应体为 `{"detail": "..."}`。调用未注册工具时 HTTP 200，`reason_code` 为 `EFFECT`。
