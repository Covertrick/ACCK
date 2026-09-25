# ACCK 术语

## 目的

给规格里已经使用的词一个固定含义，避免和旧草稿里的「五道门」或 `rejected` 混用。

## 读者

读 `docs/FEATURES.md` 和 `docs/DEVELOPMENT.md` 的人。

## 范围

只收现行规格里的词。不给未实现的沙箱、Redis 或 OAuth 造术语。

## 内容

| 词 | 含义 |
|----|------|
| Document | 一份订单 JSON。有 `doc_id`、单调 `version`、`content_hash` |
| Task | 一次协作。只写一个文档。状态只有 `running` 和 `aborted` |
| PatchIntent | 一次提交。含 `base_version`、1 到 32 条 op、幂等键、`evidence`、`effect_ids` |
| op | `add`、`remove`、`replace`、`test` 之一。空指针和带 `value` 的 `remove` 是 `SCHEMA` |
| 幂等命中 | 同一订单、同一 Agent、同一键、同一 ops 哈希。返回原结果，不写账本，不写轨迹 |
| 幂等冲突 | 同一键但 ops 哈希不同。响应 `IDEMPOTENCY_MISMATCH`，不插入第二行账本 |
| 行锁 | `SELECT documents … FOR UPDATE`。版本检查和成功写入在同一事务 |
| 检查顺序 | 功能文档第 3 节的 11 步。不是「合约 → 版本 → 效应 → 预算」 |
| abort | 冲突策略。版本落后时 `OCC`，任务仍为 `running` |
| refresh_and_replan | 冲突策略。未达上限则交回当前订单并增加 `replan_used`。达到上限则 `aborted`，码仍是 `OCC` |
| merge_if_disjoint | 冲突策略。路径互不为前缀则按当前版本继续。相交则与 `abort` 相同 |
| ToolEffect | 写类工具的真实调用记录。状态只有 `executed`、`committed`、`compensated` |
| READ | 每次真实调用，只写轨迹，不写效应表 |
| 补偿 | 对仍是 `executed` 且注册了补偿工具的效应调用 `refund` 一类工具。键为 `compensate:{effect_id}`。不占工具预算 |
| cassette | 轨迹。`step_id` 从 1 起。种类是 `llm`、`tool`、`commit`。不写 `input` |
| LIVE | 真实执行并记录 |
| STRICT | 只比较 `input_hash`。相等则用存下的 `output`。第一处不等即停。不改任务模式 |
| RESUME | 检查点之前只读轨迹，之后按 LIVE。由恢复接口把模式设成这个值 |
| Checkpoint | 成功提交之后写下的订单版本、下一 `step_id`、已用 token 和工具次数 |
| Registry | 进程内五类插件的名字表。同名第二次注册失败 |
| DocumentSpec | 插件。提供初值、角色路径，以及 patch 之后的文档约束 |
| no_kernel | 评测模式。整份写回 Agent 读到的旧文档，不进 `Kernel.commit` |
| contract_only | 评测模式。只做结构、路径、文档约束和账本。不做版本策略、去重、批准、预算、补偿、检查点 |
| full_kernel | 评测模式。功能文档的全部门 |
| success | 评测行符合该模式的期望。不是 `decision=commit` |
| silent_corruption | 最终订单与全内核不同，且该模式没有拒绝码 |
| ModelCascade | 只出现在评测报告里的桶。无内核且没有拒绝码时使用。不是接口错误码 |
| MCP | 同一 API 进程上的 `POST /mcp`。只有 `tools/list` 和 `tools/call`。请求只有 `method` 和 `params` |
| 五道门 | 旧说法。检查顺序是 11 步，不用这个词代替 |
| 协作层 | `collab/`。把目标、订单和上一轮拒绝交给模型，得到下一轮动作。不写订单 |
| 决定 JSON | 模型的一轮输出。`action` 为 `read`、`tool`、`commit`、`handoff` 或 `stop` |
| handoff | 角色结束前交给对方的一句话和 `claims`。只进 `llm` 轨迹 |
| goal | 共同目标在任务上，角色目标在绑定上。共同目标为空则不启动协作循环 |
| parallel_first_commit | 两个角色都先看版本 0，各决定一次提交，再先写规划者 |
