---
name: codelearn
description: "Code-learning / code-reverse-engineering mode. Use when the user says they are learning a codebase or asks to explain, walk through, trace, or interpret existing code — e.g. '我在学习', '请帮我解读', '帮我分析这个流程', '这段代码怎么走', 'learn/understand this code'. Produces timestamped Markdown study notes under mylearn/ containing Mermaid sequence diagrams, explicit core data structures, and a single concrete sample data (样例数据) that threads through every diagram. READ-ONLY on existing source: never modifies existing logic code so the repo stays easy to sync with upstream."
---

# CodeLearn — 代码学习 / 解读模式

把「学习现有代码」的问答沉淀为可复用的图文笔记。核心目标：**看得懂、跟得动、能同步上游**。

## 何时启用

当用户表达「在学习 / 想弄懂 / 帮我解读 / 帮我梳理流程 / 这段代码怎么走的」等意图时启用。
典型触发词：`我在学习`、`请帮我解读`、`学习一下`、`梳理一下流程`、`这段代码怎么走`、`learn`、`trace`、`walk me through`。

不确定时问一句即可，不要默认全量输出。

## 铁律（务必遵守）

1. **只读现有代码**：绝不修改 `src/`、`extension/`、`test/`、`tools/`、`third_party/` 等任何现有源码与逻辑。
   - 目的是让本地仓库保持干净，方便随时 `git pull` 同步主仓库。
   - 需要「记录」的产物一律写到 `mylearn/`，不改主干。
2. **所有学习产物只放在仓库根目录的 `mylearn/` 下**，不散落到其它目录。
3. 每次学习问答 → **生成一个独立的 `.md` 文件**，而不是只在对话里回答。
4. 画图统一用 **Mermaid**（时序图 / 类图 / 流程图等），要求清晰、直观、能一眼看懂。

## 工作流程

1. **定位代码**：用 `rg` / `read` 找出与问题相关的源码（文件 + 行号）。先读懂再输出。
2. **选一个样例数据**：构造一个最小但完整的真实例子（输入 + 期望输出），它是全文的「主线」。
3. **画时序图**：按样例数据，把调用链从上到下画出来，每个关键步骤标注该样例数据此刻的取值。
4. **画数据结构图**：用 classDiagram 表达核心对象及其关系，字段与源码对应。
5. **写代码走读**：把样例数据沿着源码逐段解释，标注 `file:line`。
6. **落盘**：写到 `mylearn/` 下的新 `.md` 文件（命名见下）。
7. **对话里回一句摘要**：给出结论 + 生成的文件路径，正文以文件为准。

> 不要在 `mylearn/` 之外创建任何文件；不要改 `.gitignore` 等已跟踪文件（需要忽略产物时用本地 `.git/info/exclude`）。

## 输出文件的硬性要求（三条缺一不可）

1. **时序图（Sequence Diagram）**：讲程序代码流程时，必须用 Mermaid `sequenceDiagram` 把执行链路画出来。
2. **核心数据结构 / 核心数据对象**：必须显式列出（名称、职责、关键字段、定义位置），并用 `classDiagram` 表达关系。
3. **样例数据贯穿**：画时序图 / 对象图 / 流程图时，**必须**用一个具体样例数据把它串起来，不能泛泛而谈。
   - 先给出样例（表、值、SQL、期望结果等），后面每张图都复用同一份数据。
   - 图里的消息、字段、分支都要带上这个样例的真实取值。

详细模板见 `references/answer-template.md`，Mermaid 用法见 `references/mermaid-cheatsheet.md`。

## 文件命名与归档

- 目录：仓库根目录 `mylearn/`
- 文件名：`YYYYMMDD-HHmmss-<slug>.md`
  - `<slug>`：问题主题的短横线小写英文/拼音，如 `physical-plan-seq`、`binder-walk`。
  - 时间取当前本地时间，保证唯一且可排序。
- 每新增一个文件，**在 `mylearn/README.md` 的「目录」里追加一行**（时间 + 一句话 + 相对链接）。
- 若 `mylearn/` 不存在则创建；`README.md` 不存在则按 `references/answer-template.md` 末尾的索引格式创建。

## 输出语言

- 默认使用用户提问的语言（通常是中文），术语保留英文原名（如 `PhysicalOperator`、`Binder`）。
- 代码引用用 `文件:行号` 标明，便于回查。

## 图表质量要求

- 时序图参与者默认不超过 ~10 个；过多就按阶段拆成多张图。
- 消息文字优先用「动词 + 样例值」，例如 `execute(chunk=[a=1,b=2])`。
- 图与文字要一一对应；图里出现的步骤，正文要能对上。
- 宁可多一张小图，也不要一张谁都看不清的大图。

## 反例（不要这样）

- ❌ 只贴大段源码解释，没有时序图。
- ❌ 画图但用 `foo()`、`value1` 这种占位符，不接样例数据。
- ❌ 读代码时顺手改了两行逻辑「为了看清」——禁止。
- ❌ 回答只留在对话里，没有生成 `mylearn/*.md`。
