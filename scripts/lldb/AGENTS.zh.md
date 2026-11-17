# 供智能体使用的 LLDB 辅助脚本

在调查 DuckDB 代码或失败的 sqllogictest 时，可在 LLDB 内使用这些 Python 脚本。它们提供 SQL 级导航，并使 C++ 值更易于检查。
在修改相关辅助脚本之前，请阅读其 README 和实现。

## 启动会话

从仓库根目录运行 LLDB，以便测试路径和测试数据正确解析。
使用根 AGENTS.md 中推荐的 `reldebug` 构建开始：

```sh
lldb -- build/reldebug/test/unittest test/sql/order/test_limit.test
```

在 LLDB 提示符下导入所需的辅助脚本：

```lldb
command script import scripts/lldb/sqllogictest_breakpoints/sql_break.py
command script import scripts/lldb/pointer_print/pointer_print.py
command script import scripts/lldb/print_array/print_array.py
command script import scripts/lldb/filter_boundschecks/filter_checks.py
```

这些是 LLDB 插件，不是独立的 Python 程序。导入仅应用于当前
调试器会话；无需修改 `~/.lldbinit`。如果 LLDB 从另一个目录启动，
请使用绝对脚本路径。

SQL 辅助脚本需要 unittest 可执行文件、已停止的进程，以及线程 1 上的 sqllogictest
帧。它们使用 `test/sqlite/sqllogic_command.cpp` 中的 `query_break`。
如果该钩子没有断点位置，或所需局部变量被优化掉，
请使用 `make debug` 和 `build/debug/test/unittest` 进行调查。
打印和步进辅助脚本也可用于 DuckDB shell 可执行文件。

## 查找 C++ 断点背后的 SQL

导入 `sql_break.py` 后，在第一个 SQL 钩子处停止：

```lldb
breakpoint set --name query_break --one-shot true
run
sql_current_statement
```

`sql_current_statement` 打印测试文件、行号、语句类型、连接、SQL
以及活动循环值。当 sqllogictest 调用者仍在线程 1 的栈上时，
它也可以从更深的 C++ 断点工作。
使用 `sql_current_statement --json` 将这些字段捕获为 JSON 对象，
而不是解析显示文本。

使用以下命令在测试中移动：

```lldb
sql_next_statement
sql_next_matching_statement --kind statement_error
sql_next_matching_statement --file test/sql/join --kind query --loop i=3
sql_next_matching_statement --connection con2
sql_next_matching_statement --sql "JOIN" --kind query
```

选择与正在调试的测试匹配的过滤器。支持的过滤器有：

- `--file <substring>` 和 `--line <n>`。
- `--line-min <n>` 和 `--line-max <n>` 表示包含范围；不要将它们与 `--line` 组合使用。
- `--kind query|statement|statement_ok|statement_error`。
- `--connection <name>` 用于精确匹配连接名称。
- `--sql <substring>` 用于对展开后的 SQL 进行区分大小写的匹配。
- `--loop <name>` 或 `--loop <name>=<value>`；可重复以添加多个循环约束。

过滤器以 AND 组合。选择行过滤器时，请使用 `sql_current_statement`
报告的位置。

对于重复停止，请安装持久规则并前进到它：

```lldb
sql_watch_statement --kind query --loop i=3
sql_list_watches
sql_next_watch
sql_delete_watch
```

`sql_next_watch <id>` 选择一个规则；省略 ID 会考虑所有已安装
规则。`sql_delete_watch <id>` 移除一个规则，省略 ID 会移除最近
添加的规则，`sql_delete_watch all` 移除所有规则。

SQL 监视是断点规则，不是内存监视点。普通的 `continue`
不会在它们上面停止。`sql_next_statement`、`sql_next_matching_statement` 和
`sql_next_watch` 会临时自动继续其他断点，并在之后恢复其
先前的设置。一旦定位到所需的 SQL，请使用普通的
`continue` 或步进，配合 C++ 断点进行调查。

这些辅助脚本假定线程 1 正在执行测试；选择工作线程
不会改变这一假设。参见 [sqllogictest_breakpoints/README.md](sqllogictest_breakpoints/README.md)。

## 检查指针和数组

导入 `pointer_print.py` 会启用 `duckdb` 类型类别，并将 `p` 别名到
`duckdb-p`。使用它直接检查 DuckDB 智能指针：

```lldb
duckdb-p my_unique_ptr
duckdb-p some_state.shared_buffer
duckdb-p /x my_optional_ptr
```

格式化器会显示指向类型和地址，暴露其子项，并将空指针渲染为 `nullptr`。它读取包装器存储，而不依赖
发出的 `operator*()` 或 `.get()` 符号。它支持 DuckDB 的 unique、shared、
optional、buffer、arena 以及列出的 unsafe 指针类型。
当您需要直接使用 LLDB 的 expression 命令时，请使用 `expression -- <expr>`。
有关支持的类型，请参见 [pointer_print/README.md](pointer_print/README.md)。

在所选帧中使用 `print_array` 查看指向多个元素的原始指针：

```lldb
print_array result_sel.sel_vector 8
print_array row_ids count
print_array "data_ptr + offset" "count - offset"
```

指针必须非空，且大小必须求值为正整数。
包含空格的参数请加引号。在已知有效元素数量范围内选择较小的大小：
该辅助脚本无法检查分配的边界。
参见 [print_array/README.md](print_array/README.md)。

## 步过包装器检查

导入 `filter_checks.py` 会立即为小型
`optional_ptr`、`unique_ptr`、`shared_ptr` 和 `vector` 辅助帧添加步进避免模式。

```lldb
duckdb-step-avoid-show
duckdb-step-avoid-disable
duckdb-step-avoid-enable
```

在调查包装器或其检查本身时，请禁用这些添加。
禁用会恢复启用添加之前保存的正则表达式。
参见 [filter_boundschecks/README.md](filter_boundschecks/README.md)。

## 更改辅助脚本

将命令语法和导入副作用记录在相应的 README 中。
保留基于 `MODULE_NAME` 的回调注册，以便脚本可以重命名。
在 LLDB 内验证行为：仅进行 Python 语法检查并不能实际运行 LLDB 的
API、表达式求值、格式化器或断点回调。对于 SQL 导航
更改，请使用小型 sqllogictest，并验证匹配、监视删除以及其他断点自动继续设置的恢复。

从仓库根目录运行集成测试套件：

```sh
python3 scripts/lldb/tests/run_tests.py --unittest build/debug/test/unittest \
  --coverage-dir /tmp/duckdb-lldb-coverage
```

该套件需要支持 Python 的 LLDB、C++ 编译器，以及在被调试器下启动
本地进程的权限。请使用具有可解析 `query_break`
符号和可检查局部变量的构建。有关覆盖率详细信息和运行器选项，请参见 [tests/README.md](tests/README.md)。
