# ADR 0003：STRICT 只比较 input_hash

## 目的

说明严格重放为什么不对着真实模型或真实工具再跑一遍，也不把整份输入另存一列。

## 读者

实现 `replay/player.py` 和演示第 3 步的人。

## 范围

只覆盖 `docs/FEATURES.md` 第 9 节和 cassette 表。轨迹种类仍是 `llm`、`tool`、`commit`。

## 内容

### 状态

已接受。

### 背景

重放要证明「用当时的输入，会取回当时记下的输出」。模型输出不稳定。若 STRICT 再调用真实模型或真实工具，同一次轨迹会因为输出不同而失败，而且 `send_receipt` 这类不可逆工具会被再执行一次。规格禁止严格重放时的真实调用。

cassette 的列是 `input_hash` 和 `output`，没有单独的输入正文列。能比较的只有哈希。

### 决定

STRICT 按 `step_id` 从小到大走已有轨迹。对每一步，用这次尝试的输入做与 `content_hash` 相同的 canonical JSON，再取 SHA-256，与存下的 `input_hash` 比较。

相等则采用该行的 `output`，不调用插件，不调用模型。第一处不等或缺少该步，返回 `REPLAY_DIVERGENCE`，带上 `step_id`、`kind`、期望哈希和实际哈希，然后停止。不改订单，不改 `tasks.mode`。

`LIVE` 才负责把当时的输入哈希和输出写进行。除幂等命中外，拒绝的提交也写 `kind=commit`，这样演示里的 `UNAUTHORIZED_PATH` 也会留在轨迹里。

### 后果

哈希相同但当时输出本身就错了，重放会原样返回那个输出。STRICT 保证的是轨迹一致，不是业务结果变正确。

篡改第 2 步的 `input_hash` 后，重放停在 `step_id=2`。第一条步骤的编号是 1。

共同目标和角色目标存在任务表上，不写 `kind=input`。内核走查不经过协作层，前两步之后重放时轨迹是 3 条 `commit`，`steps` 为 3。协作演示的轨迹另含 `llm`。
