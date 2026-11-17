本文件夹包含 DuckDB 核心的源码树。下面简要概述各个组件。

# Parser（解析器）
解析器位于 [parser](https://github.com/duckdb/duckdb/tree/main/src/parser) 文件夹。这是任何进入 DuckDB 的查询的入口点。DuckDB 使用 Postgres 的解析器（[libpg_query](https://github.com/lfittl/libpg_query)）。使用该解析器解析查询后，token 会被转换为基于 `SQLStatements`、`Expressions` 和 `TableRefs` 的自定义解析树表示。

# Planner（规划器）
规划器位于 [planner](https://github.com/duckdb/duckdb/tree/main/src/planner) 文件夹。规划器负责接收 `Parser` 从查询字符串中提取的 token，并将其转换为逻辑查询计划。该计划表示为节点类型为 `LogicalOperator` 的树。

# Optimizer（优化器）
优化器位于 [optimizer](https://github.com/duckdb/duckdb/tree/main/src/optimizer) 文件夹。优化器接收 `Planner` 生成的逻辑查询计划，并将其转换为逻辑等价但（希望）执行更快的逻辑查询计划。所执行的优化示例包括谓词下推、表达式重写和连接排序。同时执行基于代价和基于规则的优化。

# Execution（执行）
执行层位于 [execution](https://github.com/duckdb/duckdb/tree/main/src/execution) 文件夹。执行层首先接收 `Optimizer` 产生的逻辑查询计划，并将其转换为由 `PhysicalOperators` 组成的物理查询计划。随后使用基于推送的执行模型执行这些 `PhysicalOperators`。

# Catalog（目录）
Catalog 管理位于 [catalog](https://github.com/duckdb/duckdb/tree/main/src/catalog) 文件夹。Catalog 跟踪数据库中包含的表、schema 和函数。在规划阶段，`Binder` 使用 `Catalog` 将符号（例如 "table_name"）解析为数据库中实际存在的表和列。

# Storage（存储）
数据库存储组件位于 [storage](https://github.com/duckdb/duckdb/tree/main/src/storage) 文件夹。存储组件负责管理实际的物理数据，既包括内存中的数据，也包括磁盘上的数据。执行层在需要访问基表数据（例如执行基表扫描）或需要更新数据库中存储的信息（例如作为 `INSERT` 或 `UPDATE` 命令的一部分）时，会使用存储组件。

# Transaction（事务）
事务管理位于 [transaction](https://github.com/duckdb/duckdb/tree/main/src/transaction) 文件夹。事务管理器管理所有当前打开的事务，并负责处理 `COMMIT` 或 `ROLLBACK` 命令。
