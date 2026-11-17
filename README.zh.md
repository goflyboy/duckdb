<div align="center">
  <picture>
    <source media="(prefers-color-scheme: light)" srcset="https://duckdb.org/images/designmanual/DuckDB_inline-lightmode.svg">
    <source media="(prefers-color-scheme: dark)" srcset="https://duckdb.org/images/designmanual/DuckDB_inline-darkmode.svg">
    <img alt="DuckDB logo" src="https://duckdb.org/images/designmanual/DuckDB_inline-lightmode.svg" height="100">
  </picture>
</div>
<br>

<p align="center">
  <a href="https://github.com/duckdb/duckdb/actions"><img src="https://github.com/duckdb/duckdb/actions/workflows/Main.yml/badge.svg?branch=main" alt="Github Actions Badge"></a>
  <a href="https://discord.gg/tcvwpjfnZx"><img src="https://shields.io/discord/909674491309850675" alt="discord" /></a>
  <a href="https://github.com/duckdb/duckdb/releases/"><img src="https://img.shields.io/github/v/release/duckdb/duckdb?color=brightgreen&display_name=tag&logo=duckdb&logoColor=white" alt="Latest Release"></a>
</p>

## DuckDB

DuckDB 是一个高性能的分析型数据库系统。它被设计得快速、可靠、可移植且易于使用。DuckDB 在您的应用程序内部以进程内方式运行，因此无需管理单独的服务器，也没有外部依赖。在需要时，它还可以读取远程文件、查询其他数据库，并连接到其他 DuckDB 实例。它提供了丰富的 SQL 方言，其支持远超基础 SQL：任意及嵌套相关子查询、窗口函数、排序规则（collations）以及复杂类型（数组、结构体、映射）。它的[友好 SQL](https://duckdb.org/docs/current/sql/dialect/friendly_sql.html) 扩展增加了让查询更简短、更易编写的语法。

DuckDB 提供[独立的 CLI 应用](https://duckdb.org/docs/current/clients/cli/overview)，并为 [Python](https://duckdb.org/docs/current/clients/python/overview)、[R](https://duckdb.org/docs/current/clients/r)、[Java](https://duckdb.org/docs/current/clients/java)、[Rust](https://duckdb.org/docs/current/clients/rust)、[Go](https://duckdb.org/docs/current/clients/go)、[Wasm](https://duckdb.org/docs/current/clients/wasm/overview) 以及[更多语言](https://duckdb.org/docs/current/clients/overview)提供客户端，并与 [pandas](https://duckdb.org/docs/current/guides/python/sql_on_pandas) 等软件包深度集成。

有关使用 DuckDB 的更多信息，请参阅 [DuckDB 文档](https://duckdb.org/docs/current/)。

## 读写任意数据

DuckDB 可以读写多种文件格式，包括 [CSV](https://duckdb.org/docs/current/data/csv/overview)、[Parquet](https://duckdb.org/docs/current/data/parquet/overview)、[JSON](https://duckdb.org/docs/current/data/json/overview) 和 [Excel](https://duckdb.org/docs/current/core_extensions/excel)，既可以直接从本地磁盘读取，也可以从 S3、HTTP(S)、Azure 和 Google Cloud Storage 等远程端点读取。对于其中许多格式（包括 CSV、Parquet 和 JSON），加载数据只需在 FROM 子句中引用该文件即可：

```sql
SELECT * FROM 'myfile.csv';
SELECT * FROM 'myfile.parquet';
```

DuckDB 同样支持开放式湖仓（lakehouse）格式。[DuckLake](https://ducklake.select/) 将表元数据存储在 SQL 数据库中而不是元数据文件里，从而无需单独的 catalog 服务器，同时仍提供 ACID 事务、快照和时间旅行（time travel）能力。DuckDB 还支持 [Apache Iceberg](https://duckdb.org/docs/current/core_extensions/iceberg/overview) 和 [Delta Lake](https://duckdb.org/docs/current/core_extensions/delta) 格式。更多信息请参阅我们的[数据导入](https://duckdb.org/docs/current/data/overview)章节。

## 连接其他数据库

DuckDB 可以挂载（attach）其他数据库系统并就地查询它们，因此您可以在不同引擎之间读取、连接和移动数据，而无需单独的 ETL 步骤。它支持 [PostgreSQL](https://duckdb.org/docs/current/core_extensions/postgres/overview)、[MySQL](https://duckdb.org/docs/current/core_extensions/mysql) 和 [SQLite](https://duckdb.org/docs/current/core_extensions/sqlite) 的原生协议。

DuckDB 实例之间还可以通过 [quack 协议](https://duckdb.org/docs/current/quack/overview)相互连接，该协议可将一个 DuckDB 实例变为服务器，供其他 DuckDB 客户端通过 HTTP 挂载。

## 可扩展

DuckDB 拥有模块化的[扩展机制](https://duckdb.org/docs/current/extensions/overview)：核心扩展随 DuckDB 一起打包，而社区扩展和第三方扩展可以在运行时安装并加载。扩展可以增加文件格式、数据库连接器，以及新的函数、类型等。

## 安装

如果您想安装 DuckDB，请参阅[我们的安装页面](https://duckdb.org/docs/installation/)获取说明。

## SQL 参考

文档中包含[SQL 介绍与参考](https://duckdb.org/docs/current/sql/introduction)。

## 开发

进行开发时，DuckDB 需要 [CMake](https://cmake.org)、Python 3 以及兼容 `C++17` 的编译器。在根目录下运行 `make` 即可编译源码。开发时，可使用 `make debug` 构建未优化的调试版本。在做出修改后，您应运行 `make unit` 和 `make allunit` 来验证您的版本工作正常。如需测试性能，可以运行 `BUILD_BENCHMARK=1 BUILD_TPCH=1 make`，然后在根目录下执行 `./build/release/benchmark/benchmark_runner` 来运行若干标准基准测试。基准测试的详细信息见我们的[基准测试指南](benchmark/README.zh.md)。

另请参阅我们的[构建指南](https://duckdb.org/docs/current/dev/building/overview)和[贡献指南](CONTRIBUTING.zh.md)。

## 支持

请参阅[支持选项](https://ducklabs.com/support/)页面以及专门的 [`endoflife.date`](https://endoflife.date/duckdb) 页面。
