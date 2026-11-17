# 扩展补丁
本目录中的补丁用于缓解引入会破坏树外扩展兼容性的 DuckDB 更改的过程。从 git URL 安装的扩展可以自动应用本目录中找到的补丁。应使用 APPLY_PATCHES 标志显式启用此功能。例如，
假设我们的扩展配置如下：

```shell
duckdb_extension_load(spatial
    GIT_URL https://github.com/duckdb/duckdb_spatial
    GIT_TAG f577b9441793f9170403e489f5d3587e023a945f
    APPLY_PATCHES
)
```
在此示例中，下载 spatial 扩展时，`.github/patches/extensions/spatial/*.patch` 中的所有补丁
将被自动应用。

请注意，显式启用 APPLY_PATCHES 标志的原因是让阅读扩展配置的开发者更容易
发现补丁的存在。因此，如果设置了 `APPLY_PATCHES` 但 `.github/patches/extensions/<ext>/*.patch` 中
没有补丁，补丁机制实际上会失败。

# 工作流
想象引入了一个破坏与扩展 X 兼容性的 DuckDB 更改。其工作流如下：

### PR #1：对 DuckDB 的破坏性更改
1. 将破坏性更改提交到 DuckDB
2. Git clone 扩展。
3. 在 DuckDB 仓库中，转到 .github/config/extensions/your_extension.cmake
4. 如果 duckdb_extension_load 中未设置 APPLY_PATCHES，请设置它。
5. 在克隆的扩展中，检出 your_extension.cmake 文件中的 GIT_TAG。
6. 在克隆的扩展中，应用 .github/patches/extensions 中找到的现有补丁
    ```bash
    git apply [path to duckdb]/.github/patches/extensions/[your_extension]/fix.patch
    ```
7. 对扩展进行必要的更改。
8. 在克隆的扩展中生成 diff，并将其转发到原始的 fix.patch 位置（如果目录不存在则创建）：
    ```bash
    git diff > [path to duckdb]/.github/patches/extensions/[your_extension]/fix.patch

9. 使用描述性名称将补丁提交到 `.github/patches/extensions/[your_extension]/*.patch`

> [!TIP]
> 如果您所做的更改非常小，可以跳过接下来的两个步骤。（PR #2、PR #3）
>
> 接下来的两个步骤适用于以下场景：
>
> 1. 您是扩展开发者，并且更改很大，或者补丁已累积，您想升级扩展。
> 2. 您不是扩展开发者，但对扩展的更改很大，因此应遵循接下来的两个步骤，这样扩展开发者升级扩展时就不会那么繁琐。


### PR #2：对扩展 X 的补丁
- 将 `.github/patches/extensions/x/*.patch` 中的（所有）补丁应用到扩展 X。

### PR #3：在 DuckDB 中更新扩展 X
- 删除 `.github/patches/extensions/x/*.patch` 中的补丁
- 从配置中删除 `APPLY_PATCHES` 标志
- 更新配置中扩展的哈希
