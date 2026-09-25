# ACCK 数据库

## 目的

规定表、索引、事务边界和空库初始化。`store/schema.sql` 落地时必须与本文的 SQL 一致。

## 读者

实现 `store/` 和回答隔离级别、索引、迁移的人。

## 范围

只覆盖现行提交、工具、轨迹和评测会碰到的表。不引入 Redis。不写连接池的容量数字。

## 内容

PostgreSQL 16。隔离级别用默认的 `READ COMMITTED`。

### 表与索引

主键和唯一约束本身就是索引。额外索引只加一条：账本按 `doc_id` 查询并按 `created_at`、`id` 排序。

| 名字 | 服务的访问 |
|------|------------|
| `documents` 主键 `doc_id` | 行锁与按订单读取 |
| `tasks` 主键 `task_id` | 按任务读取 |
| `tasks.doc_id` 唯一 | 一文档一个任务 |
| `role_bindings` 主键 `(task_id, agent_id)` | 提交时查角色 |
| `ledger` 主键 `id` | 同时间的稳定排序 |
| `ledger` 唯一 `(doc_id, agent_id, idempotency_key)` | 幂等查找 |
| `ledger (doc_id, created_at, id)` | `GET /v1/ledger?doc_id=` |
| `checkpoints` 主键 `(task_id, seq)` | 取最大序号 |
| `cassette` 主键 `(task_id, step_id)` | 按步回放，以及 `step_id >=` 删除 |
| `effects` 唯一 `(task_id, tool_name, idempotency_key)` | 同键短路；左端 `task_id` 供补偿扫描 |

```sql
CREATE TABLE documents (
  doc_id text PRIMARY KEY,
  version int NOT NULL,
  content jsonb NOT NULL,
  content_hash text NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE tasks (
  task_id text PRIMARY KEY,
  doc_id text NOT NULL UNIQUE REFERENCES documents(doc_id),
  mode text NOT NULL,
  status text NOT NULL,
  conflict_policy text NOT NULL,
  replan_limit int NOT NULL DEFAULT 3,
  replan_used int NOT NULL DEFAULT 0,
  token_limit int NOT NULL,
  token_used int NOT NULL DEFAULT 0,
  tool_call_limit int NOT NULL,
  tool_calls_used int NOT NULL DEFAULT 0,
  deadline_at timestamptz NOT NULL,
  cassette_next int NOT NULL DEFAULT 1,
  goal text NOT NULL DEFAULT ''
);

CREATE TABLE role_bindings (
  task_id text NOT NULL REFERENCES tasks(task_id),
  agent_id text NOT NULL,
  role text NOT NULL,
  goal text NOT NULL DEFAULT '',
  PRIMARY KEY (task_id, agent_id)
);

CREATE TABLE ledger (
  id bigserial PRIMARY KEY,
  task_id text NOT NULL,
  doc_id text NOT NULL,
  agent_id text NOT NULL,
  idempotency_key text NOT NULL,
  ops_hash text NOT NULL,
  evidence jsonb NOT NULL DEFAULT '{}',
  base_version int NOT NULL,
  result_version int,
  decision text NOT NULL,
  reason_code text NOT NULL,
  ops jsonb NOT NULL,
  hash_before text NOT NULL,
  hash_after text,
  effect_ids jsonb NOT NULL DEFAULT '[]',
  trace_id text,
  span_id text,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (doc_id, agent_id, idempotency_key)
);

CREATE INDEX ledger_doc_order ON ledger (doc_id, created_at, id);

CREATE TABLE checkpoints (
  task_id text NOT NULL REFERENCES tasks(task_id),
  seq int NOT NULL,
  doc_version int NOT NULL,
  cassette_next int NOT NULL,
  token_used int NOT NULL,
  tool_calls_used int NOT NULL,
  PRIMARY KEY (task_id, seq)
);

CREATE TABLE cassette (
  task_id text NOT NULL REFERENCES tasks(task_id),
  step_id int NOT NULL,
  kind text NOT NULL,
  name text NOT NULL,
  input_hash text NOT NULL,
  output jsonb NOT NULL,
  PRIMARY KEY (task_id, step_id)
);

CREATE TABLE effects (
  effect_id text PRIMARY KEY,
  task_id text NOT NULL,
  tool_name text NOT NULL,
  idempotency_key text NOT NULL,
  effect_class text NOT NULL,
  status text NOT NULL,
  args_hash text NOT NULL,
  result jsonb,
  trace_id text,
  span_id text,
  UNIQUE (task_id, tool_name, idempotency_key)
);
```

`status` 只写 `running` 或 `aborted`。`cassette_next` 是下一个 `step_id`，第一条为 1。`tasks.goal` 是共同目标，`role_bindings.goal` 是角色目标。交接句不另建表，留在 `cassette.output`。工具合约不入库。支付桩不入库。

### 事务

一次 `Kernel.commit` 使用一个事务，并在其中 `SELECT documents … FOR UPDATE`。这个事务里包含：幂等判断、路径与预算与效应引用、版本策略、更新 `documents`、插入 `ledger`、更新 `effects`、插入 `checkpoints`、插入本次 `commit` 轨迹并推进 `cassette_next`。

拒绝时订单行不变，但仍提交账本和轨迹（幂等命中除外；幂等冲突不插入第二行账本）。任务不存在时不开启这个写事务，直接 404。

`no_kernel` 不进入该事务。它覆盖 `documents.content`，不写拒绝账本。

补偿、工具调用、重放、恢复各自使用自己的事务，不嵌在提交事务里面。恢复时的删除是 `DELETE FROM cassette WHERE task_id = $1 AND step_id >= $2`。

每个请求使用一个连接。事务提交或回滚后归还。不配置池容量。

### 初始化与迁移

空库执行一次上文 SQL，即 `store/schema.sql`。本版没有增量迁移。表结构以本文为准；尚无需要保留的线上数据时，改的是这份 SQL，而不是升级脚本。

演示与测试用的订单初值见 `docs/FEATURES.md` 第 2 节。`version` 从 0 开始。空库初始化就是本版的迁移策略。以后若要保留已有数据再改表，另写 ADR，不在本版引入 Alembic。
