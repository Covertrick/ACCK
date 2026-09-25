# ADR 0006：模型留在协作层

## 目的

说明为什么真实模型要参与规划者与收银的决定，但不能进入 `Kernel.commit` 的 11 步。

## 读者

实现 `collab/`、写课程报告或准备讲这个仓库的人。

## 范围

只覆盖 `docs/FEATURES.md` 第 14 节。不改变第 3 节的检查顺序，也不把 STRICT 改成再调用模型。

## 内容

### 状态

已接受。

### 背景

若模型只在套件里弹出一段事先写好的工具或提交 JSON，共享状态的冲突、越权和重复扣款都与模型无关。这样只能说明锁和账本，说明不了两个 Agent 的协作是否可靠。

若把模型调用放进 `Kernel.commit`，提交事务会跟着提示词和采样变化。STRICT 也无法在不再次调用模型的情况下对上 `input_hash`。

### 决定

模型只由协作层调用。每一轮它输出决定 JSON，在 `read`、`tool`、`commit`、`handoff`、`stop` 中选一个。自然语言目标存在任务和角色上。拒绝、以及仍可重规划的 `OCC`，把原因和当前订单交回模型，由它重写下一轮动作。

`Kernel.commit` 不调用模型，不解析目标文本。`kernel` 不引用 `collab`。交接句写入 `kind=llm` 的轨迹，不写入订单。

`order_v1` 和内核单测使用脚本步骤，不启动协作循环。`demo/story.py` 和 `agent_v1` 在 `ACCK_LLM=live` 时调用 OpenAI 兼容接口；否则按已录下的决定 JSON 回放。STRICT 仍然只比较 `input_hash` 并返回存下的 `output`。

### 后果

课程报告的主表是 `agent_v1` 在三种模式下的任务是否成功、首次非法率、修订是否成功、话和状态是否一致。内核十例仍用来说明这些写入为什么被挡住。

换模型或提示词只改 `collab/prompts.py` 和 `LlmClient`，不改 `commit.py`。
