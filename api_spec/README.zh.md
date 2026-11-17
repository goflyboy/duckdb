# C API 规范

本目录保存 DuckDB C API 的声明式规范。已提交的头文件由它生成：

- `src/include/duckdb.h`，C API 头文件
- `src/include/duckdb_extension.h`，C 扩展的头文件
- `src/include/duckdb/main/capi/extension_api.hpp`，引擎侧函数
  指针结构体

不要手动编辑这三个文件。编辑此处的 YAML 并重新生成。

## 布局

- `v1/`：规范。每个模块一个目录（query、value、appender 等），
  另有 `metadata.yaml`（前缀、原始类型、生命周期状态、已知 DuckDB
  版本）和 `common/`（共享枚举、结构体、回调）。
- `v1/extension/duckdb_extension.h.in`：扩展头文件的模板。
  其成员顺序就是扩展 ABI。生成器会根据规范验证每个
  成员，并在标记注释处追加新函数。
  `extension_api.hpp` 由它派生，因此两者保持同步。
- `pyproject.toml`：生成环境。`generate` 依赖
  组固定了 capigen（生成器，https://github.com/duckdb/capigen）和
  格式化工具。`uv.lock` 记录了确切的解析结果。

## 重新生成

安装一次环境，然后重新生成：

    make generate-files-deps
    make generate-files

或直接运行：`./scripts/capi_v1_regen.sh`。使用 uv 而非 pip：
`uv run --project api_spec --group generate ./scripts/capi_v1_regen.sh`。
需要 Python 3.12 或更新版本。当已提交的头文件与规范不匹配时，
CI 会失败，因此请在推送前重新生成。

## 添加函数

在模块 YAML 中声明它，并添加一个 `lifecycle` 条目，指明它随哪个
DuckDB 版本发布。如果 `metadata.yaml` 的 `versions` 中还没有该版本，
请将其添加进去。然后重新生成。该函数会出现在 `duckdb.h` 中，以及
扩展结构体的 unstable 部分。生成文件的 diff 是审查的一部分。

## 版本管理

`metadata.yaml` 中的 `schema_version` 必须与已安装的
capigen 兼容（主版本相同，规范次版本至多为工具次版本）。要升级 capigen：
更新 `pyproject.toml` 中的固定版本，运行 `uv lock --project api_spec`，
重新生成，并将头文件 diff 与固定版本一起提交。

## 编辑器 schema 验证

此处的 YAML 会根据固定版本 capigen 包中附带的 JSON Schema 进行验证。
将编辑器指向它们，以便在编辑时获得内联错误和补全。
由于 schema 来自固定安装，您的编辑器始终检查生成器所使用的
确切 schema 版本。没有任何内容被内置或提交，schema 随固定版本移动。

解析 schema 目录（因机器而异，不要提交它；更改固定版本后重新运行）：

    uv run --project api_spec --group generate python -c \
        "import importlib.resources as r; print(r.files('capigen') / 'schema')"

使用 pip 时，请使用安装了 `generate` 组的解释器运行同一单行命令。

在任何由 yaml-language-server 支持的编辑器中映射两个 schema（VS Code 配合
Red Hat YAML 扩展、Neovim yamlls、Emacs、Sublime、Helix）：

- `<schema-dir>/metadata.schema.json` 验证 `api_spec/*/metadata.yaml`
- `<schema-dir>/module.schema.json` 验证更深一层的模块文件，
  `api_spec/*/*/*.yaml`

这些 glob 相对于工作区，且从不重叠。在 VS Code 中，该映射
是 `yaml.schemas` 设置；请将其保存在用户设置中，或保存在
`.git/info/exclude` 中列出的 `.vscode/settings.json` 中，因为该路径
因机器而异。在 Neovim 中，同样的映射放在 yamlls
的 `settings.yaml.schemas` 表中。

编辑器验证只是便利，不是关卡。capigen 在重新生成期间会使用这些相同的
schema 验证每个文件，而 CI 会因生成的漂移而失败，因此无论编辑器设置如何，
无效的规范都无法落地。
