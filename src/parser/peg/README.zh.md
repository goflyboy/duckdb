# DuckDB PEG 解析器语法

本文档描述 DuckDB 用于解析 SQL 的 PEG（解析表达式语法，Parsing Expression Grammar）系统，包括语法格式、构建过程和转换器代码生成。

## 语法格式

语法规则定义在 `src/parser/peg/grammar/statements/` 下的 `.gram` 文件中。每个文件通常对应一种 SQL 语句类型（例如 `select.gram`、`insert.gram`、`alter.gram`）。共享规则和宏位于 `common.gram`。

### 规则定义

规则使用 `<-` 运算符定义：

```
RuleName <- RuleDefinition
```

### 字面量

关键字和运算符使用单引号字符串匹配：

```
'CREATE'  'TABLE'  'SELECT'  ','  '('  ')'
```

默认情况下，关键字字面量的匹配不区分大小写。

### 选择（有序备选）

`/` 运算符从左到右尝试备选。第一个成功的匹配获胜（PEG 语义——无歧义）：

```
DistinctOrAll <- DistinctKeyword / AllKeyword
LiteralExpression <- StringLiteral / NumberLiteral / ConstantLiteral
```

**重要**：由于 PEG 使用有序选择，更具体的备选必须放在一般备选之前。例如，像 `YEAR TO MONTH` 这样的复合 interval 类型必须位于简单的 `YEAR` 关键字之前，否则解析器会贪婪地消耗 `YEAR` 并在 `TO MONTH` 上失败。

### 序列

一个接一个列出的元素构成序列——所有元素必须按顺序匹配：

```
CastExpression <- CastOrTryCast Parens(CastArguments)
ShowTables <- ShowOrDescribe 'TABLES' 'FROM' QualifiedName
```

### 可选（`?`）

`?` 运算符使元素可选（零个或一个）：

```
CreateStatement <- 'CREATE' OrReplace? Temporary? CreateStatementVariation
SelectClause <- 'SELECT' DistinctClause? TargetList?
```

### 重复（`*` 和 `+`）

- `*` —— 零次或多次
- `+` —— 一次或多次

```
IsExpression <- IsDistinctFromExpression IsTest*
SearchedCaseExpression <- 'CASE' CaseWhenThen+ CaseElse? 'END'
```

### 使用括号分组

括号对子表达式分组，但不创建命名规则：

```
Interval <- (YearKeyword 'TO' MonthKeyword) /
    (DayKeyword 'TO' HourKeyword) /
    YearKeyword /
    MonthKeyword
```

### 参数化规则（宏）

规则可以接受参数，充当可复用模板。`common.gram` 中定义了两个内置宏：

```
List(D) <- D (',' D)* ','?
Parens(D) <- '(' D ')'
```

用法：

```
Parens(List(Expression))                        -- (expr, expr, ...)
Parens(DistinctOrAll? List(FunctionArgument)?)   -- 允许嵌套
List(BaseTableName)                              -- table1, table2, ...
```

### 负向前瞻（`!`）

`!` 运算符断言某个模式在当前位置不匹配（不消耗输入）：

```
PlainIdentifier <- !ReservedKeyword <[a-z_]i[a-z0-9_]i*>
```

**注意**：负向前瞻会被解析，但目前被匹配器忽略（见 `peg_parser.cpp` 中的 `FIXME`）。
请改用关键字列表和标识符匹配器来限制规则接受哪些关键字。

### 特殊 token

某些规则由专用匹配器处理，而不是通过语法展开。它们在 `compiled_grammar.cpp` 中
（`GENERATED RULE OVERRIDES` 标记之间）注册为终结规则覆盖，并在
`scripts/parser/grammar_types.yml` 的 `matcher_rule_overrides` 部分配置。它们包括：

- `Identifier` / `ReservedIdentifier` —— 通用标识符匹配
- `CatalogName`、`SchemaName`、`TableName`、`ColumnName`、`IndexName`、`SequenceName`、`TableFunctionName`、
  `TypeName`、`PragmaName`、`SettingName`、`CopyOptionName` 及其 `Reserved*` 变体 —— 上下文感知标识符
- `ReservedKeyword` —— 任何保留关键字
- `StringLiteral` —— 单引号字符串字面量
- `NumberLiteral` —— 数字字面量
- `OperatorLiteral` —— 运算符符号
- `EndOfInput` —— 语句结束

### 注释

以 `#` 开头的行是注释：

```
# Expression precedence levels
# LEVEL 1 (Lowest)
Expression <- LogicalOrExpression
```

### 关键字

关键字定义在 `src/parser/peg/grammar/keywords/` 下的列表文件中：

| 文件 | 描述 |
|------|------|
| `reserved_keyword.list` | 不能用作标识符（例如 `SELECT`、`FROM`） |
| `unreserved_keyword.list` | 可以用作标识符（例如 `ABORT`、`ABSOLUTE`） |
| `column_name_keyword.list` | 允许作为列名 |
| `func_name_keyword.list` | 允许作为函数名 |
| `type_name_keyword.list` | 允许作为类型名 |
| `bare_label_keyword.list` | 允许作为不带 `AS` 的列别名（例如 `SELECT 1 name`） |

关键字必须恰好出现在其中一个类别列表中；`inline_grammar.py` 会拒绝违规。

`bare_label_keyword.list` 与这些类别正交，遵循 Postgres 的 `bare_label_keyword`：每个条目还必须
出现在某个类别列表中，并且只有其中列出的未保留关键字或列名关键字才能用作不带 `AS` 的
别名。可以跟在表达式或类型后面的关键字被有意排除（例如 `DAY` 等 interval 单位、`FILTER` 和 `OVER` 等函数
后缀、`VARYING` 和 `WITHOUT` 等类型后缀，以及 `EXCLUDE` 等星号修饰符），因此 `SELECT d + 1 day` 是语法错误，而不是名为 `day` 的列。新的未保留关键字应
添加到列表中，除非它们可以尾随表达式。

## 构建语法

修改 `.gram` 文件、关键字列表或任何生成器配置文件后，重新生成语法和
转换器产物：

```bash
make parser-grammar
```

此目标将 Python 依赖（PyYAML）安装到格式化 virtualenv 中，并运行
`scripts/parser/build_grammar.sh`，它按顺序执行以下步骤：

1. `scripts/parser/inline_grammar.py --grammar-file` —— 将所有 `.gram` 文件和关键字列表合并为
   `inlined_grammar.gram`（用于调试）。
2. `scripts/parser/inline_grammar.py` —— 生成 `inlined_grammar.hpp`（以字节数组嵌入的语法）和
   `keyword_map.cpp`（关键字查找表）。
3. `scripts/parser/generate_transformer_trampoline.py --write` —— 生成转换器进程、规则 ID 和
   匹配器配置块（见下文）。
4. `make format-parser-grammar` —— 格式化生成的 C++ 文件。

当只需要重新生成转换器产物时，也可以直接使用 `--write` 运行转换器生成器。
不带 `--write` 时，它会打印将要生成的内容，而不是修改文件。

### 生成的文件

| 文件 | 生成器 | 描述 |
|------|--------|------|
| `src/include/duckdb/parser/peg/inlined_grammar.gram` | `inline_grammar.py` | 完整合并语法（所有 `.gram` 文件 + 关键字） |
| `src/include/duckdb/parser/peg/inlined_grammar.hpp` | `inline_grammar.py` | C++ 头文件：`const unsigned char INLINED_PEG_GRAMMAR[]` |
| `src/parser/peg/keyword_map.cpp` | `inline_grammar.py` | C++ 关键字类别查找映射 |
| `src/parser/peg/transformer/transform_generated_trampoline.cpp` | `generate_transformer_trampoline.py` | `Initialize*`/`Finalize*` 转换进程和 frame-ops 注册表 |
| `src/include/duckdb/parser/peg/transformer/peg_transformer.hpp` | `generate_transformer_trampoline.py` | `GENERATED TRANSFORM RESULT TYPES` 和 `GENERATED TRAMPOLINE RULES` 标记之间的声明 |
| `src/include/duckdb/parser/peg/peg_rule_id.hpp`、`src/parser/peg/peg_rule_id.cpp` | `generate_transformer_trampoline.py` | `PEGRuleId` 枚举，每个内置规则一个常量，以及名称查找 |
| `src/parser/peg/compiled_grammar.cpp` | `generate_transformer_trampoline.py` | `GENERATED RULE OVERRIDES` 标记之间的终结规则覆盖 |
| `src/parser/peg/matcher_factory.cpp` | `generate_transformer_trampoline.py` | `GENERATED PACKRAT MEMOIZED RULES` 标记之间的 Packrat 记忆化规则列表 |
| `src/parser/peg/transformer/peg_transformer_factory.cpp` | `generate_transformer_trampoline.py` | `GENERATED COLLAPSIBLE RULES` 和 `GENERATED EXPRESSION DEPTH RULES` 标记之间的可折叠规则列表和表达式深度规则列表 |

**不要手动编辑生成的文件或生成的块**——它们会被构建脚本覆盖。仅部分生成的
文件在生成的块周围包含 `START GENERATED ... ` / `END GENERATED ...` 标记注释；标记外的代码是手写的。

## 实现转换器规则

每条产生值的语法规则都需要一个转换器，将其解析结果转换为 DuckDB AST 节点
（`SQLStatement`、`QueryNode`、`ParsedExpression` 等）或中间辅助类型。

大部分解析结果管道是生成的。对于语法规则 `Foo`，最多有两部分：

| 函数 | 编写者 | 用途 |
|------|--------|------|
| `InitializeFooTrampoline` / `FinalizeFooTrampoline` | 生成 | 显式转换栈上的两阶段转换进程：`Initialize` 调度子节点，`Finalize` 使用它们的结果调用类型化主体并装箱返回值 |
| `TransformFoo(transformer, typed children...)` | 手写 | 语义主体：从已转换的子节点构建 AST 节点 |

仅由字面量组成的规则，或其所有备选都返回相同类型的选择规则，会端到端生成，无需手写主体。

生成的转换器代码通过 `PEGRuleId` 而不是名称引用子规则。语法扩展在运行时
添加的规则在定义它们的编译语法中获得超过 `PEGRuleId::BUILTIN_COUNT` 的 ID。

### 生成器配置

`scripts/parser/grammar_types.yml` 驱动 `generate_transformer_trampoline.py`：

- 类型类别将语法规则映射到其 C++ 返回类型（`unique_ptr<SQLStatement>`、`QualifiedName` 等）。
  任何被引用为生成规则子节点的规则都必须在此处有类型。
- `overrides` 列出具有唯一返回类型的规则。
- `excluded_rules` 列出生成进程时跳过的仅语法规则。
- `matcher_rule_overrides` 列出由专用匹配器处理的特殊 token（见上文）。
- `packrat_memoized_rules`、`collapsible_rules`、`expression_depth_nesting_rules` 和 `expression_depth_chain_rules`
  配置匹配器和转换器：哪些规则被记忆化，哪些转换在规则其余部分未匹配任何内容时原样返回其操作数，
  以及哪些规则计入 `max_expression_depth`。

`scripts/parser/transformer_trampoline.yml` 驱动逐规则进程生成。未列出的规则会获得
生成的进程。列出的规则要么是 `excluded`、`forward`（委托给单个子节点），要么是
`manual` / `manual_finalize` 并带有手写的 `init` / `finalize` 函数。

### 添加或更改规则

1. 在相关的 `.gram` 文件中添加或编辑规则。
2. 将规则的返回类型添加到 `grammar_types.yml`（如果是仅语法则添加到 `excluded_rules`）。
3. 运行 `make parser-grammar`。生成器会打印剩余的手动步骤，包括仍需手写的任何
   类型化主体的确切签名。
4. 在匹配的 `transform_*.cpp` 文件中实现主体（例如 `describe.gram` 中的规则放在
   `transform_describe.cpp` 中），然后再次运行 `make parser-grammar`，以便声明拾取它。

生成的转换器代码通过 `PEGRuleId` 而不是名称引用子规则。语法扩展在运行时
添加的规则在定义它们的编译语法中获得超过 `PEGRuleId::BUILTIN_COUNT` 的 ID。

### 转换器实现模式

手写主体按语法顺序接收已转换的子节点；字面量 token 被跳过。对于
`ShowTables <- ShowOrDescribe 'TABLES' 'FROM' QualifiedName`：

```cpp
unique_ptr<QueryNode> PEGTransformerFactory::TransformShowTables(PEGTransformer &transformer,
                                                                 const ShowType &show_or_describe,
                                                                 const QualifiedName &qualified_name) {
	auto showref = make_uniq<ShowRef>();
	showref->show_type = ShowType::SHOW_FROM;
	showref->SetSchemaName(qualified_name.Name());
	// ... 构建并返回 AST 节点
}
```

调用它的生成进程如下所示（位于 `transform_generated_trampoline.cpp`）：

```cpp
void PEGTransformerFactory::InitializeShowTablesTrampoline(PEGTransformer &transformer,
                                                           GeneratedTransformProcess &process) {
	auto &list_pr = process.parse_result.Cast<ListParseResult>();
	process.ReserveChildSlots(2);
	process.PushChild({transformer.GetRule(PEGRuleId::QualifiedName), list_pr.GetChild(3)}, 1);
	process.PushChild({transformer.GetRule(PEGRuleId::ShowOrDescribe), list_pr.GetChild(0)}, 0);
}

arena_ptr<TransformResultValue>
PEGTransformerFactory::FinalizeShowTablesTrampoline(PEGTransformer &transformer, GeneratedTransformProcess &process) {
	auto show_or_describe = process.TakeResult<ShowType>(0);
	auto qualified_name = process.TakeResult<QualifiedName>(1);
	auto result = TransformShowTables(transformer, show_or_describe, qualified_name);
	return transformer.MakeResult<unique_ptr<QueryNode>>(std::move(result));
}
```

可选子节点以 `optional<T>`（或可能为 null 的 `unique_ptr<T>`）形式到达，重复子节点以 `vector<T>` 形式到达。
只有生成器无法处理的少数规则（列在 `excluded_rules` 下，或在 trampoline 配置中为 `manual`）
仍使用下面的类型手动解包解析结果。

### 解析结果类型

| 语法构造 | 解析结果类型 | 访问模式 |
|----------|--------------|----------|
| 序列（`A B C`） | `ListParseResult` | `list_pr.Child<T>(index)`、`list_pr.GetChild(index)`、`GetChildren()` |
| 选择（`A / B`） | `ChoiceParseResult` | `choice_pr.GetResult()`（匹配的备选） |
| 可选（`A?`） | `OptionalParseResult` | `opt_pr.HasResult()`、`opt_pr.GetResult()` |
| 重复（`A+` / `A*`） | `RepeatParseResult` | `repeat_pr.GetChildren()` |
| 标识符 | `IdentifierParseResult` | `.identifier`（`Identifier`） |
| 字符串字面量 | `StringLiteralParseResult` | `.result`（字符串）、`.ToExpression()`（ParsedExpression） |
| 数字字面量 | `NumberParseResult` | `.number`（字符串） |
| 关键字 | `KeywordParseResult` | `.keyword`（字符串） |
| 运算符 | `OperatorParseResult` | `.operator_token`（字符串） |
| 输入结束 | `EndOfInputParseResult` | —— |

处理原始解析结果时，`PEGTransformerFactory::ExtractParseResultsFromList` 和 `ExtractResultFromParens` 会解包 `List(D)` 和
`Parens(D)` 宏。

### 注册

所有转换都通过一个表注册：`transform_generated_trampoline.cpp` 中的 `GeneratedTransformFrameOps()`
将每个规则名称映射到其 `Initialize*`/`Finalize*` 对，`peg_transformer_factory.cpp` 中的 `PEGTransformerFactory` 构造函数
遍历该表，将进程附加到每个规则，并附带其可折叠和
表达式深度配置。

无法生成进程的规则（`PivotStatement`、`TableRef`、`WithClause` 等）在
`transformer_trampoline.yml` 中列为 `manual` 或 `manual_finalize`，并带有其手写 `init` / `finalize` 函数的名称。
这些函数位于匹配的 `transform_*.cpp` 文件中，并出现在同一个表中，因此它们没有单独的
注册步骤。

## 从扩展扩展语法

扩展可以在运行时修改语法，而无需触及 `.gram` 文件。相关头文件位于
`src/include/duckdb/parser/`。

### GrammarExtension 和 GrammarChange

`GrammarExtension`（`grammar_extension.hpp`）是一组命名的、可组合的 `GrammarChange`
（`grammar_change.hpp`），当解析器被编译时（`compiled_grammar.cpp` 中的 `CompiledGrammar::Create`）应用于内置语法之上。可用的更改：

| 更改 | 效果 |
|------|------|
| `AddRule(definition, transform_process)` | 添加新规则，可选地为其提供转换进程 |
| `ReplaceRule(definition, transform_process)` | 替换现有规则 |
| `AddChoice(rule, choice)` / `PrependChoice(rule, choice)` | 向选择规则追加或前置一个备选 |
| `RemoveChoice(rule, find_cursor)` / `ReplaceChoice(rule, choice, find_cursor)` | 移除或替换一个备选 |
| `SetTransformProcess(rule, transform_process)` | 将转换进程附加到现有规则 |
| `AddTerminalRuleOverride(rule, matcher_factory)` | 用自定义 `Matcher` 替换规则 |

选择可以引用内置规则，因此 `GrammarChange::AddChoice("UnreservedKeyword", "'ANSWER'")` 会添加一个关键字，
`GrammarChange::PrependChoice("SelectAtom", "PipeSelectAtom")` 会将新的语句形式挂接到 `SELECT`。

使用 `GrammarExtension::Register(db, make_shared_ptr<MyGrammarExtension>())` 注册扩展，通常从
扩展的 load 函数中调用。已注册的扩展由 `duckdb_grammar_extensions()` 列出，默认不激活；
连接通过以下方式启用它们：

```sql
SET active_grammar_extensions = ['pipe_query_syntax'];
```

为添加规则编写的转换针对上述原始解析结果类型，因为生成器只
在树内语法上运行。有关添加管道语法（`FROM tbl |> WHERE ... |> SELECT ...`）的完整示例，请参见
`test/extension/loadable_grammar_extension_demo.cpp`；有关同时添加自定义匹配器的最小示例，请参见
`test/api/test_grammar_extension.cpp`。

### DialectExtension

`DialectExtension`（`dialect_extension.hpp`）是一个命名的 SQL 方言，通过
`DialectExtension::Register(config, make_shared_ptr<MyDialect>())` 注册。已注册的方言由
`duckdb_dialects()` 列出，并通过 `SET current_dialect = 'name'` 为每个连接选择；内置方言是 `duckdb`。
方言子类重写 `ApplyGrammarChanges` 以向基础语法添加、替换或拼接规则（与 `GrammarExtension` 相同的
操作），并且还可以通过 `CreateTokenizer`、`CreateKeywordHelper` 和 `CreateMatcherFactory` 额外替换分词器、关键字辅助器和匹配器
工厂。有关可加载示例，请参见
`test/extension/loadable_dialect_extension_demo.cpp`。
