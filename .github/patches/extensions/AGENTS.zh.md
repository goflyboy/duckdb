# 树外扩展补丁

本文件适用于 `.github/patches/extensions` 中的工作。在将树外扩展适配到 DuckDB 更改、更新扩展的固定版本，或添加、更改或删除扩展补丁之前，请先阅读本文件。

## 确定补丁基线

- 在 `.github/config` 下找到扩展的 `duckdb_extension_load` 声明，并使用其确切的 `GIT_URL` 和 `GIT_TAG`。不要针对其他分支、标签或检出创建补丁。
- 从该 `GIT_TAG` 的干净检出开始工作。在进行新更改之前，按文件名字典序应用该扩展的每个现有补丁。新补丁必须基于其之前所有补丁应用后的结果。
- 补丁路径相对于扩展仓库的根目录，而不是 DuckDB 仓库。生成包含 `a/` 和 `b/` 路径的原始统一 diff，使其同时适用于 `patch -p1` 和 `git apply`。不要使用 `git format-patch`，也不要包含其邮件头。
- 每个补丁应聚焦于一项连贯的兼容性更改。排除无关的格式化、生成文件变动、构建产物和其他工作区更改。

## 命名与放置补丁

- 将补丁存储在 `.github/patches/extensions/<extension>/` 中。
- 将每个新补丁命名为 `NNNN-short-description.patch`，其中 `NNNN` 是四位、补零的序列号，例如 `0001` 或 `0012`。
- 补丁按文件名字典序应用。通常选择现有最大数字前缀之后的下一个编号，使新补丁最后应用。确保任何有依赖的补丁排在其前置补丁之后。
- 使用唯一且具描述性的文件名。不要模仿遗留的未编号名称或以 `zzz` 为前缀的名称；它们早于编号约定。
- 不要在单个扩展目录中放置 README 文件、笔记或任何其他辅助文件。补丁应用脚本会拒绝该目录中任何不以 `.patch` 结尾的条目。正因如此，本 `AGENTS.md` 位于共享的父目录中。

## 启用补丁应用

- 为扩展添加第一个补丁时，在相关的 `duckdb_extension_load` 声明中添加 `APPLY_PATCHES`。否则补丁目录会被忽略。
- 只要扩展有补丁，就保持 `APPLY_PATCHES` 存在。移除最后一个补丁时，作为同一更改的一部分移除 `APPLY_PATCHES`。`make extension-patch-check` 会拒绝没有该标志的补丁以及没有补丁的标志。
- 如果扩展有多个配置分支或加载声明，请确保每个能够获取已打补丁版本的适用声明都启用了补丁。

## 支持的本地工作流

同步工作流将现有补丁保留为配置的 `GIT_TAG` 之上的提交，并可将这些提交导出回补丁文件：

```shell
BUILD_EXTENSIONS=<extension> make sync_out_of_tree_extensions
cd extension/external/<extension>
# 进行并验证一项聚焦的更改。
git add -A
git commit -m 'NNNN-short-description.patch'
cd ../../..
EXPORT_EXTENSION_PATCHES=1 BUILD_EXTENSIONS=<extension> make sync_out_of_tree_extensions
```

- 在创建扩展的第一个补丁时，请在初始同步之前添加 `APPLY_PATCHES`。
- 提交主题必须与预期的补丁文件名完全匹配：单个词、以 `.patch` 结尾、在补丁提交中唯一，并且在字典序上晚于前一个提交主题。每个补丁使用一个提交。
- 导出会根据对应的提交重写每个补丁。请检查所有导出的文件，并确认较早的补丁没有意外更改。
- 另一种方法是：克隆 `GIT_URL`，检出 `GIT_TAG`，按排序顺序应用现有补丁，进行聚焦的更改，并将干净的 `git diff` 写入新补丁路径。

`FORCE_APPLY_PATCHES=1` 会在重新应用补丁之前硬重置并清理已同步的扩展检出。它会丢弃本地扩展工作，因此仅在确实希望这种破坏时使用。`DUCKDB_SKIP_APPLYING_PATCHES=1` 是临时的本地调试绕过手段；它不是修复，不得用于宣称补丁工作已完成。

## 更新或退役补丁

- 更改 `GIT_TAG` 时，从新的固定版本开始，并按文件名顺序重新应用所有剩余补丁。对不再能干净应用的补丁进行变基或重新生成。
- 删除其更改已包含在上游的补丁。保留剩余补丁的相对顺序，如果没有剩余补丁则移除 `APPLY_PATCHES`。
- 不要仅仅为了添加新补丁而压缩或重新生成无关的现有补丁。

## 验证更改

- 构建涉及的扩展，并针对已打补丁的检出运行其相关测试。对于已同步的检出，典型的针对性构建以 `DUCKDB_NEW_EXTENSION_BUILD=1 BUILD_EXTENSIONS=<extension> make reldebug` 开始。
- 运行 `make extension-patch-check`。
- 运行 `git diff --check`，并在完成前检查每个已更改或新生成的补丁。
