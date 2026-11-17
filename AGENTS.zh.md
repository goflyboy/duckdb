# AGENTS.md

本文件为编码智能体（coding agents）在本仓库中处理代码时提供指引。

## 概述

DuckDB 是一个高性能分析型数据库系统，被设计得快速、可靠、可移植且易于使用。它是一个分析型数据库管理系统，拥有丰富的 SQL 方言、向量化执行引擎和列式存储格式。

## 构建命令

### 基础构建
```bash
make debug                   # 构建带 sanitizer 和断言的调试版本
make reldebug                # 构建带调试符号的优化发布版本
FORCE_DEBUG=1 make relassert # 构建带 sanitizer 和断言的优化版本
```

## 测试

### 运行测试
```bash
build/reldebug/test/unittest # 快速单元测试
```

### 运行特定测试
```bash
# 运行特定测试文件
build/reldebug/test/unittest test/sql/order/test_limit.test

# 运行所有测试（包括慢速测试）
build/reldebug/test/unittest "*"
```

建议使用 `make reldebug` 和 `build/reldebug/test/unittest`，除非有充分理由使用调试构建——调试构建比 reldebug 构建慢得多。

### 使用 LLDB 调试

在使用 LLDB 调查 DuckDB 代码或失败的 sqllogictest，或修改 LLDB 辅助脚本之前，请阅读 [scripts/lldb/AGENTS.md](scripts/lldb/AGENTS.md)。其中涵盖了能感知 SQL 的断点、智能指针和数组的打印，以及如何步过包装检查。

### 限制查询时间

使用 `max_execution_time` 设置（毫秒，`0` = 不限制）中止运行过久的查询：

```sql
SET max_execution_time=5000;
SELECT count(*) FROM range(100000000) t1, range(1000) t2;
-- INTERRUPT Error: Query exceeded maximum execution time
```

该设置默认是 `LOCAL` 的；使用 `SET GLOBAL max_execution_time=...` 将其应用于所有连接。从 shell 中：

```bash
build/reldebug/duckdb -c "SET max_execution_time=5000; SELECT ..."
```

截止时间在 `ClientContext::InterruptCheck()` 调用点强制执行，因此只有执行到达这些调用点之一时才会触发。频繁检查的查询会在接近限制时中止，但规划所花的时间或不可中断的长代码段内的时间不受其约束。


### 测试文件格式
测试使用 sqllogictest 格式（`.test` 文件）。示例结构：
```sql
# name: test/sql/order/test_limit.test
# description: Test LIMIT keyword
# group: [order]

statement ok
CREATE TABLE test (a INTEGER, b INTEGER);

query I
SELECT a FROM test LIMIT 1
----
11

statement error
SELECT a FROM test LIMIT a
----
<REGEX>:Binder Error:.*not found.*
```

测试指令：
- `statement ok` - 语句应当成功
- `statement error` - 语句应当失败
- `query I` - 返回 INTEGER 列的查询
- `query II` - 返回两列的查询
- `----` - 分隔查询与预期结果
- `<REGEX>:` - 预期的错误消息模式
- `require-env VAR` - 测试需要环境变量

慢速测试应使用 `.test_slow` 扩展名而不是 `.test`。

不要在测试中添加 `PRAGMA enable_verification`——新测试中不应再使用它。

测试不得产生 DuckDB 警告（例如来自已弃用的 pragma 或设置）。`test/configs/warnings_as_errors.json` 配置（`--warnings-as-errors`）会让任何记录警告的语句失败；有意检查警告的测试应执行 `SET enable_logging=true; SET warnings_as_errors=true;` 并用 `statement error` 预期该警告。

不要在测试中循环遍历 `enable_optimizer`（或类似设置）——测试已通过 `configs/disable_optimizer.json` 配置在禁用优化器的情况下运行。

## 代码格式化

```bash
make format-fix        # 格式化所有代码（clang-format + black）
make generate-files    # 生成文件 + 格式化所有代码
```

确保在提交前运行格式化。

## 大量测试 / 让 CI 正常工作

以下是一组应按顺序运行的测试，以确保变更集通过 CI 中的大量测试。如果用户要求您修复 CI，请确保以下命令成功。

```bash
make allunit
FORCE_DEBUG=1 FORCE_ASSERT=1 make reldebug && build/reldebug/test/unittest
make test_configs
make test_vector
```


## 架构

### 查询执行流水线

```
SQL 字符串
    ↓
[PARSER] - 使用 PEG 解析器将 SQL 解析为 AST
    ↓
SQLStatement 树（ParsedExpression、TableRef 对象）
    ↓
[PLANNER/BINDER] - 将符号绑定到 catalog，创建逻辑计划
    ↓
逻辑计划（带有已绑定 Expression 的 LogicalOperator 树）
    ↓
[OPTIMIZER] - 应用基于规则和基于代价的优化
    ↓
优化后的逻辑计划
    ↓
[PHYSICAL PLAN GENERATOR] - 转换为物理算子
    ↓
物理计划（PhysicalOperator 树）
    ↓
[EXECUTOR] - 以向量化、并行流水线方式执行
    ↓
结果
```

### 核心组件

**Parser**（`src/parser/`）
- 将 SQL 字符串转换为抽象语法树（AST）
- 使用基于 PEG 的解析器
- 语法位于 `*.gram` 文件中，并使用 `scripts/parser/build_grammar.sh` 生成
- 输出：`SQLStatement`、`ParsedExpression`、`TableRef` 对象
- 关键子目录：`expression/`、`statement/`、`tableref/`、`peg/`

有关添加新语法的更多细节，请参阅位于 `src/parser/peg/README.md` 的 README。
每条新语法规则都必须有对应的转换器规则，位于 `src/parser/peg/transformer`。

**Planner**（`src/planner/`）
- 将符号绑定到 catalog 条目并解析类型
- 创建逻辑查询执行计划
- 关键类：`Binder`、`LogicalOperator`、已绑定的 `Expression` 类型
- 子目录：`binder/`、`expression/`、`subquery/`

**Optimizer**（`src/optimizer/`）
- 在不改变语义的前提下转换逻辑计划
- 应用谓词下推、连接排序、表达式重写等
- 子目录：`join_order/`、`statistics/`、`rule/`、`pushdown/`

**执行引擎**（`src/execution/`）
- 将逻辑计划转换为物理计划并执行
- 基于推送（push-based）的向量化执行模型
- 按批次处理数据（通常每批 2048 行）
- 关键子目录：`operator/`（scan、join、filter、aggregate 等）、`expression_executor/`

**存储**（`src/storage/`）
- 管理持久化数据存储和缓冲区管理
- 基于块的存储，带压缩
- 包含用于持久性的 WAL（预写日志）
- 子目录：`buffer/`、`compression/`、`checkpoint/`、`table/`

**Catalog**（`src/catalog/`）
- 表、schema、函数、类型等的元数据管理
- 数据库元数据的唯一事实来源
- 关键类：`Catalog`、`CatalogEntry`、`SchemaCatalogEntry`

**事务管理器**（`src/transaction/`）
- 基于 MVCC 的 ACID 事务管理
- 协调对数据的并发访问
- 关键文件：`transaction_manager.cpp`、`undo_buffer.cpp`、`wal_write_state.cpp`

**并行执行**（`src/parallel/`）
- 带任务调度的多线程执行
- 基于流水线的并行
- 关键文件：`executor.cpp`、`pipeline_executor.cpp`、`task_scheduler.cpp`

**函数**（`src/function/`）
- 内置函数实现
- 类型：`scalar/`、`aggregate/`、`table/`、`window/`、`pragma/`

### 目录结构

```
/duckdb
├── src/                   # 核心 C++ 源码
│   ├── include/duckdb/   # 公共头文件
│   ├── parser/           # SQL 解析
│   ├── planner/          # 逻辑规划
│   ├── optimizer/        # 查询优化
│   ├── execution/        # 物理执行
│   ├── storage/          # 数据存储
│   ├── catalog/          # 元数据管理
│   ├── transaction/      # 事务管理
│   ├── parallel/         # 并行化
│   ├── function/         # 内置函数
│   ├── common/           # 共享工具与类型
│   └── main/             # 数据库/连接管理
├── extension/            # 树内扩展（parquet、json、icu 等）
├── test/                 # 测试框架与测试用例
│   ├── sql/             # SQL 回归测试（.test 文件）
│   └── api/             # C/C++ API 测试
├── tools/                # Shell、sqllogictest、Swift
├── benchmark/            # 基准测试套件（TPC-H、TPC-DS 等）
├── scripts/              # 构建与实用脚本
└── third_party/          # 第三方依赖
```

## 扩展

DuckDB 支持两种类型的扩展：

**树内扩展**（位于 `extension/` 目录）：
- 扩展位于树内
- 完整列表见 `.github/config/in_tree_extensions.cmake`
- 代码可以直接编辑并提交到仓库。

**树外扩展（Out-of-Tree）**：
- 扩展位于单独的 git 仓库
- 完整列表见 `.github/config/out_of_tree_extensions.cmake`
- 当必须进行更改时，必须将其放在 `.github/patches` 中的补丁文件里
- 在适配树外扩展、更改其固定版本或修改 `.github/patches/extensions` 之前，请阅读并遵循 `.github/patches/extensions/AGENTS.md`。
- 某些树外扩展（例如 `iceberg`）需要 vcpkg。如果构建失败并提示 `VCPKG_TOOLCHAIN_PATH is not set`，请遵循 `extension/README.md` 中的"Installing vcpkg"。

使用扩展构建：
```bash
# 构建所有扩展
BUILD_ALL_EXT=1 make
# 构建特定扩展
DUCKDB_EXTENSIONS='json;icu' make
```

## 关键开发模式

### 数据流
- **向量化处理**：数据按列式批次处理（而非逐行），通常每批 2048 行
- **Vector 类**：表示一个列式数据批次
- **ColumnBinding**：贯穿规划/执行阶段的列唯一标识符 `(table_index, column_index)`

### 表达式类型
- `ParsedExpression` - 来自解析器，未绑定
- `Expression` - 已绑定类型信息
- `ExpressionExecutor` - 表达式的向量化执行

### 内存管理
- 独占所有权优先使用 `unique_ptr<T>`
- 仅在必要时使用 `shared_ptr<T>`
- `optional_ptr<T>` 用于可空引用，`reference<T>` 用于非空引用
- 绝不使用裸指针

### 类型系统
- `LogicalType` - 抽象数据类型表示
- 类型提升规则位于 `src/function/cast_rules.cpp`
- 通过扩展系统支持自定义类型

### 常见模式
- **访问者模式**：用于树遍历（例如 `LogicalOperatorVisitor`、`ExpressionIterator`）
- **工厂模式**：用于对象创建的 `Deserialize()` 方法
- **类层次结构**：基类如 `*Operator`、`*Entry`、`*Expression` 及带类型的子类

## Vector API

向量携带自身的大小，元素访问通过带类型的迭代器（读取）和带类型的写入器（写入）进行。**旧式向量循环已弃用，不得在新代码中使用：**

```cpp
// 不要这样做 - 旧式模式：ToUnifiedFormat + 手动 sel/validity 索引
auto idx = vdata.sel->get_index(i);
if (!vdata.validity.RowIsValid(idx)) {
	result_validity.SetInvalid(i);
}
result_data[i] = data[idx] + 1;
```

```cpp
// 要这样做 - 迭代器 + 写入器
auto input_values = input.Values<int64_t>();
auto writer = FlatVector::Writer<int64_t>(result, count);
for (auto entry : input_values) {
	if (!entry.IsValid()) {
		writer.WriteNull();
		continue;
	}
	writer.WriteValue(entry.GetValue() + 1);
}
```

例外：逐元素的标量函数仍应使用执行器（`UnaryExecutor`、
`BinaryExecutor`、`GenericExecutor` 等），以及编译期不知道 C++ 类型的类型擦除代码
（哈希表、排序、行布局、存储）仍使用 `ToUnifiedFormat(data)`。

头文件：`duckdb/common/vector/vector_iterator.hpp`、`duckdb/common/vector/vector_writer.hpp`。

### 读取：`Vector::Values<T>()`

适用于任何向量类型——绝不要仅为读取而调用 `Flatten()`。在循环外构建一次（它会
物化一个 `UnifiedVectorFormat`），然后迭代或索引。它不可复制；请以
`const VectorIterator<T> &` 传给辅助函数。

```cpp
auto values = vec.Values<string_t>();
for (auto entry : values) {   // 或：values[i]、values.size()、values.CanHaveNull()
	entry.IsValid();          // 每行的有效性
	entry.GetValue();         // 值（会断言有效性）；GetValueUnsafe() 跳过断言
	entry.GetIndex();         // 行索引
}
```

还有 `vec.ValidValues<T>()`（完全跳过 NULL 行）和 `vec.Validity()`（仅有效性）。

嵌套类型有迭代器特化，因此嵌套读取无需子列/偏移算术：

```cpp
// STRUCT(BIGINT, VARCHAR)
auto structs = vec.Values<VectorStructType<int64_t, string_t>>();
for (auto entry : structs) {
	entry.IsValid();                         // 顶层 struct 的 NULL
	auto a = entry.GetChildValue<0>();       // 每个子列的 ValueEntry
	entry.ForEach([&](auto &child) { ... }); // 按声明顺序遍历所有子列
}

// LIST(BIGINT) - 递归嵌套，例如 VectorListType<VectorListType<double>>
auto lists = vec.Values<VectorListType<int64_t>>();
for (auto entry : lists) {
	entry.GetListLength();
	entry.GetChildValue(idx);
	for (auto child : entry.GetChildValues()) { ... }
}

// VARIANT - 无需展开（unshredding）即可遍历 shredded 和未 shredded 的 variant
VectorIterator<VectorVariantType> variants(vec);
```

`VectorListType<T>` 迭代器不接受 `DICTIONARY_VECTOR` 源——如果列表向量可能是
字典，请先展平。

### 写入：`FlatVector::Writer<T>()`

写入器是基于推送的：按顺序写入行，写入器自行设置结果向量的大小
（无需调用 `SetSize` / `SetCardinality`）。使用 `(result, count, offset)` 重载在
偏移处追加。析构时写入器会断言恰好写入了 `count` 行——如果您有意写入更少的行，
请调用 `Truncate()`。

```cpp
auto writer = FlatVector::Writer<int64_t>(result, count);
writer.WriteValue(value);
writer.WriteNull();
```

`VectorWriter<string_t>` 额外提供 `WriteStringRef(val)`（引用而不复制到向量的堆中——
通过 `StringVector::AddHeapReference` 保持其存活）、`WriteEmptyString(len)` 和 `GetHeap()`。

嵌套写入器与迭代器镜像对应：

```cpp
// STRUCT：WriteNull() 会传播到每个子列，使子列保持同步
auto writer = FlatVector::Writer<VectorStructType<int64_t, string_t>>(result, count);
writer.WriteValue([&](auto &a_writer, auto &b_writer) { ... });
writer.ForEach([&](auto &child_writer) { ... });  // 对每个子列执行相同操作

// LIST，每行已知长度
for (auto &child_writer : list_writer.WriteList(n)) {
	child_writer.WriteValue(...);
}

// LIST，未知长度 - 按需增长，在返回的写入器离开作用域时完成
auto list = list_writer.WriteDynamicList();
list.WriteElement().WriteValue(...);
```

如果元素类型在编译期未知，`FlatVector::Writer<list_entry_t>` 有
`WriteDynamicList()`，返回 `DynamicListAppender`（`Append(source, sel, ...)` / `AppendNulls`）。
对于行无法按顺序写入的罕见情况，`FlatVector::ScatterWriter<T>(vec)` 支持
随机访问（`writer[idx] = value`、`writer.SetInvalid(idx)`）。

## 编码指南（要点）

### C++ 风格
- 使用制表符缩进，使用空格对齐
- 行宽不应超过 120 列（运行格式化工具）
- 使用 `[u]int(8|16|32|64)_t` 而不是 `int`、`long` 等
- 对于偏移/索引/计数使用 `idx_t` 而不是 `size_t`
- 对非平凡对象使用 `const` 引用
- 尽可能使用 C++11 基于范围的 for 循环
- if 语句和循环始终使用花括号
- 绝不使用 `const_cast`

### 注释约定

尽量保持注释简短。通常注释应为一短行。只有在特殊情况下注释才应超过一短行。代码应基本自描述，过多的大段注释会让代码更难阅读和理解。

避免添加与特定 issue 相关的、说明更改是如何进行的注释。例如，像"添加 +1 以修复差一错误"这样的注释与理解代码无关。此类与所解决问题相关的注释应放在 PR 描述或提交消息中，而不是代码本身。

### 命名约定
- **文件**：`snake_case`（例如 `abstract_operator.cpp`）
- **类型**：`PascalCase`（例如 `LogicalOperator`）
- **变量**：`snake_case`（例如 `chunk_size`）
- **函数**：`PascalCase`（例如 `GetChunk`）

### 类布局
```cpp
class MyClass {
public:
    MyClass();
    int my_public_variable;

public:
    void MyFunction();

private:
    void MyPrivateFunction();

private:
    int my_private_variable;
};
```

### 错误处理
- 对终止查询的错误使用异常（解析错误、表未找到、内存不足等）
- 对查询期间可恢复的错误使用返回值
- 对程序员错误（绝不由用户输入触发）使用 `D_ASSERT`
- 大量使用断言并添加清晰注释

### 测试要求
- 优先使用 sqllogictest 框架（`.test` 文件）而非 C++ 测试
- 使用不同类型测试（数值、字符串、嵌套类型）
- 测试意外/不正确的用法，而不仅仅是正常路径
- 慢速测试应使用 `.test_slow` 扩展名
- 提交 PR 前所有测试必须通过（`make allunit`）
- 追求高代码覆盖率

## 导航技巧

### 查找组件
- 入口点：`src/main/database.cpp`（DatabaseInstance）
- 查询执行协调器：`src/main/client_context.cpp`
- SQL 解析：`src/parser/parser.cpp`
- 逻辑规划：`src/planner/planner.cpp`
- 优化编排：`src/optimizer/optimizer.cpp`
- 物理计划生成：`src/execution/physical_plan_generator.cpp`
- 执行编排：`src/parallel/executor.cpp`

### 搜索代码库
- 使用 `grep` 或 `ripgrep` 搜索代码
- 函数定义通常在 `.cpp` 文件中
- 类声明在 `src/include/duckdb/` 头文件中
- 测试用例按功能位于 `test/sql/`

### 理解一项功能
1. 在 `test/sql/` 中查找测试用例以了解用法示例
2. 从 parser → planner → optimizer → execution 追踪
3. 查找对应的 `*Statement`、`*Operator`、`*Expression` 类
4. 检查 catalog 中的函数注册

### 修改生成文件
某些文件是自动生成的。修改其源文件后，运行：
```bash
make generate-files
```
这会重新生成：
- C API 绑定
- 函数注册
- 设置（Settings）
- 序列化代码
- 存储信息
- 指标枚举
- 枚举工具

## 文档

- 主文档：https://duckdb.org/docs/
- 开发文档：https://duckdb.org/dev/
- 构建指南：https://duckdb.org/docs/dev/building/overview
- 测试文档：https://duckdb.org/dev/testing

## 重要文件

- `Makefile` - 主构建配置
- `CMakeLists.txt` - CMake 配置
- `CONTRIBUTING.md` - 贡献指南
- `test/README.md` - 测试文档
- `extension/extension_config.cmake` - 扩展配置
- `scripts/format.py` - 代码格式化工具
- `scripts/generate_*.py` - 代码生成脚本
