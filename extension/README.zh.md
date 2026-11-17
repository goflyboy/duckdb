本 readme 解释 DuckDB 中有哪些类型的扩展以及如何构建它们。

# 什么是 DuckDB 扩展？
DuckDB 扩展是包含附加 DuckDB 功能的库，与主代码库分离。这些
扩展可以为 DuckDB 提供由于各种原因不能/不应存在于 DuckDB 主代码中的附加功能。
DuckDB 扩展可以通过两种方式构建。首先，它们可以静态链接到 DuckDB 的可执行文件中（duckdb cli、
unittest 二进制、benchmark runner 二进制等）。这样做会在使用这些二进制时自动使其可用。
其次，DuckDB 有扩展加载机制，可以动态加载扩展二进制文件。

# 扩展类型
DuckDB 扩展可以分为不同类型：树内扩展和树外扩展。这些类型指
扩展所在的位置以及由谁维护。

### 树内扩展
树内扩展是存在于主 DuckDB 仓库中的扩展。这些扩展被认为是 DuckDB 的基础，
和/或与 DuckDB 紧密耦合，以至于对 DuckDB 的更改预计会经常破坏它们。我们力求
将树内扩展的数量保持在最低限度，并尽可能将扩展移到树外。
### 树外扩展（OOTEs）
树外扩展存在于主 DuckDB 仓库之外的单独仓库中。将扩展移到树外的原因
各不相同。首先，将扩展移出主 DuckDB 代码库可以保持核心 DuckDB 代码更小、
更简单。其次，将扩展保持在树外可能出于许可原因很有用。

OOTE 主要有两种类型。首先是 **DuckDB 托管的 OOTE**。这些通过主
DuckDB CI 分发。这些扩展使用 DuckDB 的签名密钥签名，并由 DuckDB 团队维护。一些示例是
`sqlite_scanner` 和 `postgres_scanner` 扩展。DuckDB 托管的 OOTE 随每个
DuckDB 版本自动分发。有关此类别的当前扩展列表，请查看 `.github/config/out_of_tree_extensions.cmake`

其次是**外部 OOTE**。此类别的扩展不绑定到 DuckDB CI，而是其 CI/CD
在自己的仓库中运行。外部 OOTE 仓库的维护者负责测试、分发并
确保扩展的最新版本可用。取决于谁维护扩展，这些扩展
可能已签名，也可能未签名。

# 构建扩展
在底层，所有类型的扩展都以相同方式构建，即使用 DuckDB 的根 `CMakeLists.txt` 文件作为根 CMake 文件，
并将应构建的扩展传递给它。DuckDB 有多种方法配置要构建哪些扩展。
此外，我们可以为每个扩展配置如何构建它：例如，是仅
构建可加载扩展，还是也将扩展链接到 DuckDB 二进制文件中。在 DuckDB 中有不同的方式加载扩展，

## Makefile/Cmake 变量
指定要加载哪些扩展的最简单方法是使用 `DUCKDB_EXTENSIONS` 变量。要在构建 duckdb 时指定
构建哪些扩展，请将 extensions 变量设置为以 `;` 分隔的扩展名称列表。例如：
```bash
DUCKDB_EXTENSIONS='json;icu' make
```
选择扩展的 Make 构建会自动使用同步扩展构建路径。这包括
`DUCKDB_EXTENSIONS`、`BUILD_EXTENSIONS`、`CORE_EXTENSIONS`、`BUILD_<extension>` 简写、
`BUILD_ALL_*` 和 `EXTENSION_CONFIGS`。设置 `DUCKDB_NEW_EXTENSION_BUILD=0` 可临时使用旧版
构建路径。

`DUCKDB_EXTENSIONS` 变量会传递给 CMake 变量 `BUILD_EXTENSIONS`，也可以直接调用：
```bash
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_EXTENSIONS='parquet;icu;tpch;tpcds;fts;json'
```

### 项目拥有的扩展版本

使用同步扩展构建路径时，同步脚本和 CMake 使用相同的配置
优先级：显式 `EXTENSION_CONFIGS` 文件优先，然后是命名的 `BUILD_EXTENSIONS` 配置。
每个扩展的首次声明获胜，包括 `SOURCE_DIR` 和 `DONT_BUILD`
声明。因此，扩展项目可以提供自己的仓库 URL 和
`GIT_TAG` 哈希，而无需更改 DuckDB 检入的默认值：

```bash
EXTENSION_CONFIGS=/path/to/project/extension_config.cmake \
BUILD_EXTENSIONS='httpfs;avro;aws' make reldebug
```

`EXTENSION_CONFIGS` 是以分号分隔的列表；较早的文件优先于较晚的
文件。Make 将其作为 `DUCKDB_EXTENSION_CONFIGS` 传递给 CMake。

要拥有命名配置的整个目录，请设置 `EXTENSION_CONFIG_BASE_DIR`。例如，
以下内容从项目目录读取 `httpfs.cmake` 和 `avro.cmake`：

```bash
EXTENSION_CONFIG_BASE_DIR=/path/to/project/extension-configs \
BUILD_EXTENSIONS='httpfs;avro' make reldebug
```

默认值是 DuckDB 的 `.github/config/extensions`。自定义目录会替换该
查找位置；缺失的文件不会回退到 DuckDB 固定的版本。同步
和 CMake 都会从 DuckDB 源目录解析相对配置路径。对于由另一个仓库驱动的构建，
建议使用绝对路径。

同步脚本还接受 `--extension-configs` 和 `--extension-config-base-dir`。
CMake 接受 `-DEXTENSION_CONFIG_BASE_DIR=...` 或环境变量。构建包装器
必须将相同的设置传递给两个步骤。

直接 CMake 构建不会自动运行同步步骤。请先调用同步脚本并设置
`DUCKDB_NEW_EXTENSION_BUILD=1`，或对于包含树外扩展的构建使用 Make 包装器。

同步读取字面 `duckdb_extension_load` 声明，并遵循绝对 include 和使用
`${CMAKE_CURRENT_LIST_DIR}` 或 `${EXTENSION_CONFIG_BASE_DIR}` 的 include。它不
求值通用 CMake 变量或条件；在同步使用的配置中保持仓库 URL 和提交哈希
为字面量。这控制扩展 Git 版本，而不是 vcpkg 注册表
基线。

## Makefile 环境变量
指定构建扩展的另一种方法是使用根
`Makefile` 中定义的 `BUILD_<extension name>` 变量。例如，要构建 JSON 扩展，只需运行 `BUILD_JSON=1 make`。这些 Makevars
应为每个扩展手动添加，它们只是 DUCKDB_EXTENSIONS 变量周围的语法糖。

## 配置文件
为了更好地控制树内扩展的构建方式，应使用扩展配置文件。这些配置文件
只是由 DuckDB 的 CMake 构建包含的 CMake 文件。将搜索 4 个不同的位置
以查找配置文件：

1) 基础配置 `extension/extension_config.cmake`。此处指定的扩展将在每次构建 DuckDB 时
构建。此配置始终被加载。
2) （可选）`tools/*/duckdb_extension_config.cmake` 中的客户端特定扩展规范。这些配置指定
哪些扩展被构建并链接到每个客户端。
3) （可选）本地配置文件 `extension/extension_config_local.cmake` 您可以在此指定需要在
DuckDB 本地/自定义/开发构建中包含的扩展。此文件被 gitignore，由开发者创建。
4) （可选）传递给 `DUCKDB_EXTENSION_CONFIGS` 参数的附加配置文件。这可以用于将 DuckDB
指向机器上任意位置存储的配置文件。

DuckDB 将以相反顺序加载这些配置文件，并忽略后续对同名扩展的加载调用。
这允许通过在当地配置中提供不同的配置来覆盖扩展的基础配置。
例如，parquet 扩展被构建并静态链接到 DuckDB，因为
`extension/extension_config.cmake` 中有以下行：
```cmake
duckdb_extension_load(parquet)
duckdb_extension_statically_link(parquet)
```
现在假设我们想用自定义 parquet 扩展构建 DuckDB。我们可以通过创建
`extension/extension_config_local.cmake` 文件并添加以下内容来实现：
```cmake
duckdb_extension_load(parquet
    SOURCE_DIR /path/to/my/custom/parquet
)
```

# 构建与链接
`duckdb_extension_load` 只决定构建什么。DuckDB 目标（shell、unittest、库）静态链接什么
是单独决定的：默认情况下，配置中用
`duckdb_extension_statically_link(<extension_name>)` 列出的已构建扩展（加上使用
`LINK_CORE_EXTENSIONS=1` 构建时 `BUILD_EXTENSIONS` 中命名的扩展），或当设置了 `STATICALLY_LINK_EXTENSIONS` 时
恰好是它（例如 `STATICALLY_LINK_EXTENSIONS='core_functions;parquet;json;icu;tpch' make`）。已构建但未
链接的扩展仍然可用：自动安装来自构建自己的扩展仓库，因此它会在查询首次需要时被安装并
加载。

# 使用扩展配置文件
配置文件中的 `duckdb_extension_load` 函数用于指定应如何加载扩展。
有 3 种不同的方式。有关一些示例，请查看 `.github/config/*.cmake`。这些是
DuckDB CI 中用于选择构建哪些扩展的配置。

## 自动加载
加载扩展的最简单方法是只传递扩展名称。这会自动尝试加载扩展。
```cmake
duckdb_extension_load(<extension_name>)
```
这种 `duckdb_extension_load` 配置会在 `./extension` 和 `./extension_external` 目录中搜索
扩展，并尝试在可能时加载它们。请注意，`extension_external` 目录不存在，但应
创建并填充应构建的树外扩展。基于
[extension-template](https://github.com/duckdb/extension-template) 的扩展在放入 `extension_external` 目录时，
应能通过这种自动加载开箱即用。

## 自定义路径
当扩展位于某个路径中或其项目结构与
[extension-template](https://github.com/duckdb/extension-template) 不同时，可以使用 `SOURCE_DIR` 和 `INCLUDE_DIR` 变量告诉 DuckDB
如何加载扩展：
```cmake
duckdb_extension_load(<extension_name>
    SOURCE_DIR <absolute_path_to_extension_root>
    (INCLUDE_DIR <absolute_path_to_extension_header>)
)
```

## 远程 GitHub 仓库
也支持直接从 GitHub 仓库安装扩展。这会将扩展下载到当前
cmake 构建目录并从那里构建：
```cmake
duckdb_extension_load(postgres_scanner
    GIT_URL https://github.com/duckdb/postgres_scanner
    GIT_TAG cd043b49cdc9e0d3752535b8333c9433e1007a48
)
```

# 显式禁用扩展
因为有时您可能想覆盖其他配置设置的扩展，也可以使用 `DONT_BUILD flag` 显式禁用扩展。
这会完全禁止构建该扩展。例如，要构建不带默认启用的 parquet 扩展的 DuckDB，请在 `extension/extension_config_local.cmake` 中指定：
```cmake
duckdb_extension_load(parquet DONT_BUILD)
```
请注意这也可以通过 Makefile 完成：
```bash
DUCKDB_EXTENSIONS='tpch;json' SKIP_EXTENSIONS=parquet make
```
结果为：
```bash
...
-- Building extension 'tpch' from '/Users/sam/Development/duckdb/extensions'
-- Building extension 'json' from '/Users/sam/Development/duckdb/extensions'
-- Extensions linked into DuckDB: tpch, json
-- Extensions explicitly skipped: parquet
...
```

# 报告扩展版本

扩展可以通过 `EXT_VERSION_<NAME_UPPERCASE>` 宏报告其版本，DuckDB 将其定义为
构建时配置的扩展版本：
```cpp
std::string ParquetExtension::Version() const {
#ifdef EXT_VERSION_PARQUET
	return EXT_VERSION_PARQUET;
#else
	return "";
#endif
}
```
默认情况下，该宏会为扩展的每个源文件定义，包括它内置的任何第三方代码。
由于版本默认为 git 提交哈希——树内扩展是 DuckDB 的，否则是扩展自己的——
这会在其所有编译器命令行中放入一个每次提交都会变化的值，迫使
整个扩展为一个文件读取的字符串重新编译。

扩展通过在自己的 `CMakeLists.txt` 中声明其哪些源文件实际读取该宏来避免这一点：
```cmake
project(ParquetExtension)

# 在任何子目录或目标之前，否则它们会继承目录范围的宏定义
set_extension_version_sources(parquet parquet_extension.cpp)
```
随后只有列出的源文件会获得 `EXT_VERSION_<NAME>`，版本变化会重新编译这些文件而不是
所有内容。必须在扩展声明任何目标或添加任何子目录之前调用它，因为
那些会在创建时继承目录范围的宏定义。树外扩展也可以使用它；
它们的版本是自己的提交哈希，因此其仓库中每次提交都会变化。

# VCPKG 依赖管理
DuckDB 扩展可以使用 [VCPKG](https://vcpkg.io/en/) 管理其依赖。请查看 [Extension Template](https://github.com/duckdb/extension-template) 了解
如何在扩展中设置 vcpkg 的示例。

## 安装 vcpkg
在安装 vcpkg 之前，构建带有 `vcpkg.json` 的扩展（例如 `iceberg`）会失败并提示
`VCPKG_TOOLCHAIN_PATH is not set`。请检出 CI 使用的 vcpkg 提交（`.github/workflows/Main.yml` 中的 `vcpkg-commit`），
以便依赖版本与 CI 匹配：
```shell
git clone https://github.com/microsoft/vcpkg ~/vcpkg
git -C ~/vcpkg checkout <vcpkg-commit from .github/workflows/Main.yml>
~/vcpkg/bootstrap-vcpkg.sh -disableMetrics
export VCPKG_TOOLCHAIN_PATH=~/vcpkg/scripts/buildsystems/vcpkg.cmake
GEN=ninja CORE_EXTENSIONS='iceberg' make
```
扩展被构建为可加载扩展；使用 `duckdb -unsigned` 和
`LOAD 'build/release/extension/iceberg/iceberg.duckdb_extension'` 加载它们。

## 使用多个依赖 vcpkg 的扩展构建 DuckDB
要构建带有多个都使用 vcpkg 的扩展的 duckdb，需要一些额外步骤。这是因为每个
扩展都会为其依赖指定自己的 vcpkg.json 清单，但 vcpkg 只允许单个清单。这里的变通方法
是合并所有正在构建的扩展清单中的依赖。本仓库包含一个自动执行此合并的脚本。

### 使用 2 个依赖 vcpkg 的扩展的示例构建
例如，假设我们想创建一个 DuckDB 二进制文件，其中静态链接了两个各自使用 vcpkg 的扩展。第一步是将这两个扩展
添加到 `extension/extension_config_local.cmake`：
```cmake
duckdb_extension_load(extension_1
    GIT_URL https://github.com/example/extension_1
    GIT_TAG some_git_hash
)
duckdb_extension_load(extension_2
    GIT_URL https://github.com/example/extension_2
    GIT_TAG some_git_hash
)
```

现在要合并这两个扩展的 vcpkg.json 清单，请运行：
```shell   
make extension_configuration
```
这将在 `./build/extension_configuration/vcpkg.json` 中创建一个合并清单。

接下来，运行：
```shell
USE_MERGED_VCPKG_MANIFEST=1 VCPKG_TOOLCHAIN_PATH="/path/to/your/vcpkg/installation" make
```
这将使用合并清单安装所有必需的依赖，构建 `extension_1` 和 `extension_2`，构建 DuckDB，
最后将两个扩展链接到 DuckDB。

# 手动链接静态归档
在 CMake 之外，程序为其想要的每个扩展链接一个归档，编译生成的静态扩展加载器，
并将引擎归档放在最后：
```shell
LINK_EXTENSIONS="parquet;json" make static_extension_loader     # 写入 build/release/static_extension_loader.c
cc -I src/include main.c build/release/static_extension_loader.c libparquet_extension.a libjson_extension.a libduckdb_static.a -lstdc++
```
每个扩展归档都携带一个 describe 函数 `duckdb_extension_<name>_describe`，它填充
`duckdb_extension_descriptor`（参见 `duckdb_static_extension.h`）。加载器是纯 C：它定义
`duckdb_register_static_extensions()`，将每个 describe 函数传递给 `duckdb_register_static_extension`，这也会
将扩展从其归档中拉出。在打开数据库之前，从任何能调用 C 的语言调用它。希望这在 main 之前发生的
C++ 程序，请将 `extension/loader/static_extension_autoregister.cpp` 与
加载器一起编译到程序本身而非归档中，否则链接器会丢弃它。没有 `LINK_EXTENSIONS` 时，
加载器会注册构建中的每个扩展归档。程序也可以跳过加载器，对其自己声明的 describe 函数
调用 `duckdb_register_static_extension`。describe 函数未注册的归档不会贡献任何内容，
而已注册但归档缺失的则会导致链接失败。CMake 目标从
`link_extension_libraries` 获得所有这些。

可加载扩展是同一归档作为共享库链接并导出其入口点，当它被构建为携带自己的 DuckDB 副本时，
位于引擎归档旁边：
```shell
c++ -shared -o parquet.duckdb_extension libparquet_extension.a libduckdb_static.a -Wl,-exported_symbol,_parquet_duckdb_cpp_init
```
describe 函数在那里永远不会被采用，因为没有东西注册它。
