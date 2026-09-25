# ACCK 贡献说明

## 目的

规定改行为、改表和加测试时改哪一份文档，避免再写出互相矛盾的规格。

## 读者

改这个仓库的人。

## 范围

仓库目前以文档为准，代码尚未按开发文档落地。本文不规定分支名、代码评审人数或提交信息格式。

## 内容

行为只改 `docs/FEATURES.md`。实现步骤和测试文件只改 `docs/DEVELOPMENT.md`。表、索引和事务只改 `docs/DATABASE.md`。用例期望只改 `docs/HARNESS.md`。接口示例改 `docs/API.md` 和 `docs/openapi.yaml`。已经接受的理由改 `docs/ADR/`，并在 `docs/DECISIONS.md` 留一行。不要在功能文档里再写一篇理由。

不要把 Kubernetes、Redis、Go sidecar、OAuth、可重排的门、效应状态 `rejected` 或任务状态 `completed` 加回验收。这些在需求里是不做。

新测试对上开发文档第 8 节的文件，或对上 `docs/HARNESS.md` 已有的用例名。不要为了覆盖 `refresh_and_replan` 上限再新增一个评测用例名；该行为在 `tests/test_occ.py`。

代码落地之后，提交前运行：

```text
python -m unittest discover -s tests
```

评测：

```text
python -m harness.run --suite harness/suites/order_v1.yaml --mode full_kernel --out experiments/out
```

`commit.py` 里不写订单字段。字段、状态迁移和角色路径放在 `order` 的 `DocumentSpec`。

不规定远程仓库、分支名和评审人数。行为和代码不一致时，先改 `docs/FEATURES.md`，再改实现文档和代码，使它们与功能文档一致。
