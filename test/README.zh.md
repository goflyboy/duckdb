这是 DuckDB 的 SQL 和 C++ 单元测试

典型入口：
- POSIX 构建：`build/release/test/run`、`build/reldebug/test/run`、`build/debug/test/run`
- Windows 原生布局：`build/release/test/run.py` 或 `build/release/test/run.bat`

## 测试契约

在 SQL 测试中，测试运行器（unittest）保证这些环境变量会被设置。许多变量可在需要时覆盖。

| 环境变量 | 默认值 | 覆盖方式 | 说明 |
|----------|--------|----------|------|
| `TEST_NAME` | 例如 `test/path/filename.test` | 不适用 | |
| `TEST_NAME__NO_SLASH` | 例如 `test_path_filename.test` | 不适用 | 始终为 `TEST_NAME s@/@_@g`（保留主体后缀） |
| `TEST_ID` | 例如 `test_path_filename_test` | 不适用 | 每测试标识：完整的 `TEST_NAME` 被规范化为单个路径组件（`[A-Za-z0-9_-]` 之外的每个字符 → `_`，包括 `.`）；保留主体后缀，以便仅后缀不同的同级测试（`foo.test` 与 `foo.test_slow`）保持区分；`TEMP_DIR_ROOT` 下的 `[TEST_ID]` 路径层级 |
| `TEST_UUID` | 随机 UUID（字符串） | 不适用 | 每次调用定义 |
| `RUN_ID` | 生成的 UUID | `unittest --run-id <id>`<br/>另有 `DUCKDB_TEST_RUN_ID`、配置 `run_id` | 每次运行的标识；`[RUN_ID]` 路径层级；在一次运行的所有测试间共享 |
| `WORKING_DIR` | 例如 `/Users/me/src/duckdb` | `unittest --test-dir <path>` | 默认：duckdb 源码和构建所在位置<br/>别名：`{WORKING_DIRECTORY}` |
| `BUILD_DIR` | `{WORKING_DIR}/build/release` | 不适用 | 只读；由 unittest 路径派生<br/>别名：`{BUILD_DIRECTORY}` |
| `DATA_DIR` | `{WORKING_DIR}/data` | `unittest --data-dir <path>`<br/>另有 `DUCKDB_TEST_DATA_DIR`、配置 `data_dir` | `duckdb/data/**` 的副本，用于测试 AWS/Azure/其他 VFS 读取；可以是相对路径或绝对/远程路径（例如 `az://...`）。参见[数据目录](#数据目录)。 |
| `TEMP_DIR_ROOT` | `{WORKING_DIR}/duckdb_unittest_tempdir` | `unittest --temp-dir-root <path>`<br/>另有 `DUCKDB_TEST_TEMP_DIR_ROOT`、配置 `temp_dir_root` | 临时目录树的根；可以是本地或远程（例如 `s3://`） |
| `TEMP_DIR` | `{TEMP_DIR_ROOT}/[RUN_ID]/[TEST_ID]` | `--temp-dir-run-id {on,off}` / `--temp-dir-test-id {on,off}`（包含/省略每个层级）；`--temp-dir-destroy` 控制生命周期 | 每测试临时目录；用于测试 VFS 写入<br/>别名：`{TEST_DIR}`（已弃用的 `__TEST_DIR__` 会被 `make format-check` 拒绝） |
| `TEMP_DIR_ABSOLUTE` | `{TEMP_DIR}` 的绝对路径 | 不适用 | 当测试需要绝对路径时使用，例如 `file://` URI |
| `LOCAL_TEMP_DIR` | `{TEMP_DIR}` | `unittest --local-temp-dir-root <path>`（仅根；必须本地，并且需要远程 `TEMP_DIR_ROOT`——将本地树固定到某处而非默认位置。与不同的本地 `TEMP_DIR_ROOT` 一起使用会报错，因为它已经是自己的 `LOCAL_TEMP_DIR`；使用 `DUCKDB_TEST_TEMP_DIR_ROOT` 重定向整棵树）<br/>另有 `DUCKDB_TEST_LOCAL_TEMP_DIR_ROOT`、配置 `local_temp_dir_root` | 保证为**本地**的临时目录——当 `TEMP_DIR_ROOT` 为本地时是同一路径，当其为远程时是单独的本地树（相同的 `[RUN_ID]/[TEST_ID]` 层级，相同生命周期）。用于在面向远程根的测试中本地暂存/溢出。 |
| `LOCAL_DATA_DIR` | `{DATA_DIR}` | `unittest --local-data-dir <path>`（必须本地，并且当 `--data-dir` 本身为本地时必须与之相等）<br/>另有 `DUCKDB_TEST_LOCAL_DATA_DIR`、配置 `local_data_dir` | 保证为**本地**的数据目录；当 `DATA_DIR` 为远程时回退到 `{WORKING_DIR}/data` |
| `CATALOG_DIR` | `{TEMP_DIR}/{TEST_UUID}` | 不适用 | 每测试 catalog 目录；不保证存在（按需创建） |
| *（任意）* | 不适用 | `unittest --env-passthrough <NAME>`（可重复，或逗号分隔）<br/>另有 `DUCKDB_TEST_ENV_PASSTHROUGH`、配置 `env_passthrough` | 预注册**调用方**的环境变量，用于在每个测试中替换 `{NAME}`。现有的 `require-env` 接受已注册的值。与 `require-env` 的每测试跳过不同，已命名但缺失的变量会在启动时使整个调用失败。上表中的名称是保留名称，会被拒绝。 |

上表中的每个名称都是**保留的**：运行器会自行解析它，每次调用一次或
每个测试一次。`--env-passthrough` 和配置中的 `test_env` 条目都不能设置保留名称——命名
保留变量会在启动时使整个调用失败，而不是静默覆盖已解析的
值。请改用匹配的选项（`--data-dir`、`--temp-dir-root`、`--local-data-dir` 等）；
`test_env` 用于某个测试套件*自己*的变量。

### 环境前提条件

`require-env NAME` 在环境变量不存在时跳过整个测试。
可选的空格分隔值列表要求与任一列出的值精确、区分大小写地匹配。
`require-env-not NAME` 要求该变量不存在。
对于排除值列表，`require-env-not NAME VALUE...` 则要求
变量存在且不匹配任何列出的值。

```text
# 针对任一受支持的存储后端运行。
require-env TEST_STORAGE_BACKEND local memory
```

```text
# 此测试需要具有持久存储的后端。
require-env-not TEST_STORAGE_BACKEND memory
```

```text
# 此测试要求 VARIABLE_NAME 变量未定义。
require-env-not VARIABLE_NAME
```

两个指令使用相同的环境查找，包括配置中的 `test_env` 值，
并为存在的值注册 `{NAME}` 替换。仅要求不存在的形式
不会注册替换或环境标签。已定义但为空的变量
算作存在。值是字面 token：逗号、`!` 和 `not` 没有特殊
含义。每个指令适用于整个测试，不能出现在循环内。
现有的 `require-env NAME` 和 `require-env NAME VALUE` 测试保持其行为。

### 数据目录

`DATA_DIR` 是测试读取固定数据的位置（测试中的 `{DATA_DIR}`）。使用 `--data-dir <path>`、`DUCKDB_TEST_DATA_DIR=<path>`、`data_dir` 配置选项设置它。

- **默认（未设置）：** `{WORKING_DIR}/data`。它遵循工作目录：树外扩展测试在运行中途 `chdir` 到自己的源码目录，因此其 `DATA_DIR` 变为 `<extension>/data`——这是有意为之，因为这些测试自带 `data/` 并相对于仓库根引用它。
- **已设置，相对路径：** 按原样使用，并在读取时相对于*当前* cwd 解析，因此它同样遵循测试的 `chdir`。
- **已设置，绝对或远程**（例如 `az://…`、`s3://…`）：按原样使用，从不重新锚定——跨 `chdir` 保持稳定。**调用方有责任**使绝对 `DATA_DIR` 对运行中的每个测试有效（或排除会更改工作目录的测试）。

### 临时/测试目录

每个测试获得一个临时目录，在测试中写作 `{TEST_DIR}`。其路径最多有三层嵌套：

    TEMP_DIR_ROOT / [RUN_ID] / [TEST_ID]

- **`TEMP_DIR_ROOT`** —— 根（`duckdb_unittest_tempdir`，或 `--temp-dir-root`）；可以是本地或远程（`s3://…`）。解析时会规范化，因此 `.`/`..` 段会被折叠（词法上——不访问磁盘），并且 `TEMP_DIR_ROOT` 报告解析后的形式。相对根保持相对。空根在启动时被拒绝。
- **`RUN_ID`** —— 每次运行的隔离，在一次运行的所有测试间共享；使用 `--temp-dir-run-id` 切换（固定 id 会将多批 `unittest` 放在同一个目录下；bind-mount 会关闭它）。
- **`TEST_ID`** —— 每测试隔离；使用 `--temp-dir-test-id` 切换。按测试解析（测试运行前名称未知），并在首次使用时惰性物化。

生命周期是一种销毁处置，向下继承到每一层：`--temp-dir-destroy {never,on-success,always}`（默认 `on-success`）。创建是无条件的 `mkdir -p`，并且每一层都会记录它是否已经存在，因此销毁只回收运行本身创建的内容——包括运行根，当运行仅采用它时会保留（从不递归删除）。两者也覆盖 HOME 沙箱和 `LOCAL_TEMP_DIR` 树——`--temp-dir-destroy never` 保留运行创建的所有内容，以供检查。

**远程** `TEMP_DIR_ROOT` 会将销毁限制为 `never`：对象存储创建即写入，因此测试拥有物化过程，`unittest` 从不对远程根执行 mkdir/rm。`LOCAL_TEMP_DIR` 覆盖该配置中仍需要本地的内容——参见上文的契约表。它是一个完整的本地树（`[RUN_ID]/[TEST_ID]`，相同处置），以默认本地根或 `--local-temp-dir-root` 为根。

上述每个 `--temp-dir-*` 标志（以及 `--run-id`、`--database-destroy`、`--env-passthrough`）都是标准测试配置选项：每个也接受 `DUCKDB_TEST_<NAME>` 环境变量，并可从 `--test-config` 文件设置。CLI 优先于环境变量；稍后加载的配置文件优先于两者。

`TEMP_DIR` 下的**已加载数据库文件**有自己的处置，它是 `--temp-dir-destroy` 的独立同级选项：`--database-destroy {on,off,on-success}`（默认 `on-success`）。数据库的保留/销毁本身不是临时目录属性，因此它是一个单独的开关——使用 `off` 在保留的临时目录内保留生成的数据库文件（例如在运行后对比它们，如 `test_zero_initialize.py` 所做）；`on-success` 与 `--temp-dir-destroy` 一致，因此失败测试的数据库会保留以供检查。

**命名——读一次，然后不必再担心：**
- `{TEST_DIR}`（几乎所有测试使用的 token）和 `TEMP_DIR` 环境变量是**同一个目录**——同义词。优先使用 `{TEST_DIR}`；`{TEMP_DIR}` 也会解析到它。运行器仍接受已弃用的拼写 `__TEST_DIR__`、`__WORKING_DIRECTORY__`、`__BUILD_DIRECTORY__` 和 `${NAME}`（用于循环迭代器或环境变量 `{NAME}`），并给出警告，但 `make format-check` 会在本仓库中拒绝它们，`make format-fix` 会重写它们。
- `{TEST_DIR}`/`TEMP_DIR` **默认是相对路径**（相对于 `WORKING_DIR`）——这是有意为之，以免绝对机器路径泄漏到被比较的测试输出中。当测试确实需要绝对路径（例如 `file://` URI）时，请使用 **`TEMP_DIR_ABSOLUTE`**。
- `TEMP_DIR_ROOT` 只是根；`CATALOG_DIR` 是 `{TEST_DIR}/{TEST_UUID}`（按需创建）。

**扩展测试是唯一的例外。** 树外扩展测试在运行中途 `chdir` 到自己的源码目录，因此*相对* `{TEST_DIR}` 会针对错误的目录解析。仅对于这些测试，运行器将 `{TEST_DIR}`/`TEMP_DIR` 固定为绝对路径——在 chdir 之前捕获；其他一切保持相对。

### 并行 CSV 读取器

位于 `test/parallel_csv/test_parallel_csv.cpp` 的测试会在多种线程和缓冲区大小配置下运行并行 CSV 读取器。

### ADBC

ADBC 测试位于 `test/api/adbc/test_adbc.cpp`。要运行它们，必须在 `DUCKDB_INSTALL_LIB` 变量中提供 DuckDB 库路径。
