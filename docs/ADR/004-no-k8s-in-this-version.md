# ADR 0004：本版不做 Kubernetes

## 目的

说明为什么验收里没有 Agent Sandbox、gVisor、Kata 或 Helm。

## 读者

要把本仓库说成完整 Agent 运行平台的人。

## 范围

只重申 `docs/REQUIREMENTS.md` 的「不做」。不写沙箱接口，也不写一份未实现的 CRD。

## 内容

### 状态

已接受。

### 背景

本仓库要证明的是共享订单的提交：越权进不了文档、并发后写不会静默覆盖、同键不会二次扣款、崩溃后从最后一次成功提交继续。这些都发生在 `Kernel.commit`、效应记录和 cassette 里。

Kubernetes Agent Sandbox 解决的是进程放在哪个隔离环境里、沙箱如何预热。它不决定 `base_version` 落后时是拒绝、重规划还是按路径合并。

### 决定

本版不实现 K8s、Sandbox CRD、WarmPool、gVisor、Kata、Helm。运行方式是 `docker-compose.yml` 里的 PostgreSQL 16 和 API。API 端口 8000，数据库端口 5432。

也不把「只写设计、不实现」算进验收。需求里的验收清单没有沙箱条目。

### 后果

简历和演示应说协作提交内核，不说企业级 Agent 运行平台。换隔离方式不会改变 11 步检查顺序。

Go sidecar、Redis、Kafka、Dapr、Service Mesh、完整 A2A、OAuth 同样不做，理由相同：它们不参与提交正确性。

以后若单独立项做沙箱，新写一篇 ADR。不把 CRD 补进本版验收。
