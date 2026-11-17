
#### 克隆仓库并编译基准测试运行器

```
git clone https://github.com/duckdb/duckdb
cd duckdb
BUILD_BENCHMARK=1 BUILD_TPCH=1 make
```

#### 列出所有可用的基准测试
`build/release/benchmark/benchmark_runner --list`

#### 运行单个基准测试
`build/release/benchmark/benchmark_runner benchmark/micro/nulls/no_nulls_addition.benchmark`

输出将以 CSV 格式打印到 `stderr`，格式如下：

```
name	run	timing
benchmark/micro/nulls/no_nulls_addition.benchmark	1	0.121234
benchmark/micro/nulls/no_nulls_addition.benchmark	2	0.121702
benchmark/micro/nulls/no_nulls_addition.benchmark	3	0.122948
benchmark/micro/nulls/no_nulls_addition.benchmark	4	0.122534
benchmark/micro/nulls/no_nulls_addition.benchmark	5	0.124102
```

默认情况下，每个基准测试在一次预热运行后产生五次计时运行。您可以使用 `--timed-runs n` 覆盖此设置：

```
build/release/benchmark/benchmark_runner benchmark/micro/nulls/no_nulls_addition.benchmark --timed-runs 20
```

您还可以使用 `--out` 标志指定输出文件。这会将仅包含计时数据的内容（以换行分隔）写入该文件。

```
build/release/benchmark/benchmark_runner benchmark/micro/nulls/no_nulls_addition.benchmark --out=timings.out
cat timings.out
0.182472
0.185027
0.184163
0.185281
0.182948
```

#### 正则表达式
您还可以使用正则表达式指定要运行哪些基准测试。注意某些正则表达式字符可能被 shell 展开（例如 `*` 很可能被您的 shell 展开，因此需要适当加引号或转义）。

`build/release/benchmark/benchmark_runner "benchmark/micro/nulls/.*" `

#### 运行所有基准测试
不指定任何参数将运行所有基准测试。

`build/release/benchmark/benchmark_runner`

#### 仅解析器的微基准测试

使用 `Parser.*` 基准测试来测量不执行查询的 `Parser::ParseQuery()`。
有关工作负载、计时边界以及使用 `.github/regression/parser.csv` 进行版本比较，请参阅[解析器基准测试指南](micro/parser/README.md)。

#### 其他选项
`--info` 会给出关于基准测试的一些其他信息。

```
build/release/benchmark/benchmark_runner benchmark/micro/nulls/no_nulls_addition.benchmark --info
display_name:NULL Addition (no nulls)
group:micro
subgroup:nulls
```

`--query` 将打印基准测试所运行的查询。

```
SELECT MIN(i + 1) FROM integers
```

`--profile` 将输出查询树（美化打印），主要用于交互式使用。

```
┌─────────────────────────────────────┐
│┌───────────────────────────────────┐│
││    Query Profiling Information    ││
│└───────────────────────────────────┘│
└─────────────────────────────────────┘
SELECT MIN(i + 1) FROM integers
┌─────────────────────────────────────┐
│┌───────────────────────────────────┐│
││         Total Time: 0.176s        ││
│└───────────────────────────────────┘│
└─────────────────────────────────────┘
┌───────────────────────────┐
│    UNGROUPED_AGGREGATE    │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│          min(#0)          │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│             1             │
│          (0.03s)          │
└─────────────┬─────────────┘                             
┌─────────────┴─────────────┐
│         PROJECTION        │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│          +(i, 1)          │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│         100000000         │
│          (0.05s)          │
└─────────────┬─────────────┘                             
┌─────────────┴─────────────┐
│          SEQ_SCAN         │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│          integers         │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│             i             │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│         100000000         │
│          (0.08s)          │
└───────────────────────────┘      
```

