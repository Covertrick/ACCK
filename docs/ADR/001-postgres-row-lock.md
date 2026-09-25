# ADR 0001：用 Postgres 行锁把版本检查和写入放进同一事务

## 目的

说明为什么共享订单的并发正确性依赖 `SELECT … FOR UPDATE`，而不是另加一把锁。

## 读者

实现 `Kernel.commit` 和回答「单库怎么并发」的人。

## 范围

只覆盖本版已经定下的提交事务。不讨论跨库方案。

## 内容

### 状态

已接受。对应 `docs/FEATURES.md` 第 3 节第 4 步，以及 `docs/DATABASE.md` 的事务一节。

### 背景

两个 Agent 可以拿着同一个 `base_version` 同时提交。如果先读版本、释放锁、再写入，后写会盖掉先写，评测里的 `same_field` 就无法变成 `OCC`。

幂等键和账本也必须和这次写入同生同灭。提交回滚时，不能留下「订单没改、幂等键已经占上」或相反的窗口。

### 决定

在 `READ COMMITTED` 下对 `documents` 行执行 `SELECT … FOR UPDATE`。版本比较、patch、账本、检查点、效应改为 `committed`，都在这把行锁所在的事务里完成。后拿到锁的事务读到的是新版本，因此得到 `OCC`，而不是覆盖。

幂等键的唯一约束也在这个事务里。撞上唯一约束的一方再读已提交行：ops 哈希相同则返回原结果，不同则 `IDEMPOTENCY_MISMATCH`，不插入第二行。

不使用 Redis、咨询锁或 `SERIALIZABLE`。规格已把 Redis 列为不做。行锁加上账本里的成功 ops，足够实现三种冲突策略。

### 后果

一份订单一行。同一 `doc_id` 的提交会排队。这是本版要的正确性，不是额外的队列产品。

`no_kernel` 不进入 `Kernel.commit`，因此没有这把锁，用来复现整份覆盖。

本地 Docker、PostgreSQL 16 上，提交延迟目标是 p50 小于 50 毫秒、p95 小于 200 毫秒。超出只记入评测报告的 `latency_note`，不把用例判失败。数字的定义在 `docs/HARNESS.md`。
