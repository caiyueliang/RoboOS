# Domain Docs

工程 skills 在探索代码库时应该如何使用本仓库的领域文档。

## Before exploring, read these

- 仓库根目录的 **`CONTEXT.md`**，或者
- 如果存在仓库根目录的 **`CONTEXT-MAP.md`** — 它指向每个 context 的一个 `CONTEXT.md`。阅读与主题相关的每个文件。
- **`docs/adr/`** — 阅读涉及你即将工作领域的 ADR。在多 context 仓库中，还要检查 `src/<context>/docs/adr/` 中的 context 特定决策。

如果这些文件都不存在，**静默继续**。不要标记它们缺席；不要主动建议创建它们。生产者 skill (`/grill-with-docs`) 会在术语或决策实际确定时才会创建它们。

## File structure

单-context 仓库（大多数仓库）:

```
/
├── CONTEXT.md
├── docs/adr/
│   ├── 0001-event-sourced-orders.md
│   └── 0002-postgres-for-write-model.md
└── src/
```

多-context 仓库（根目录存在 `CONTEXT-MAP.md`）:

```
/
├── CONTEXT-MAP.md
├── docs/adr/                          ← 系统级决策
└── src/
    ├── ordering/
    │   ├── CONTEXT.md
    │   └── docs/adr/                  ← context 特定决策
    └── billing/
        ├── CONTEXT.md
        └── docs/adr/
```

## Use the glossary's vocabulary

当你在输出中命名一个领域概念时（在 issue 标题、重构提案、假设、测试名称中），使用 `CONTEXT.md` 中定义的术语。不要使用词汇表明确避免的同义词。

如果你需要的概念还不在词汇表中，这是一个信号 — 要么你发明了项目不使用的语言（重新考虑），要么存在真正的空白（为 `/grill-with-docs` 记录下来）。

## Flag ADR conflicts

如果你的输出与现有 ADR 矛盾，要明确提出而不是静默覆盖：

> _与 ADR-0007（事件溯源订单）矛盾 — 但值得重新开放，因为…_
