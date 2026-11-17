# 贡献指南

## 行为准则

本项目及其所有参与者均受[行为准则](CODE_OF_CONDUCT.zh.md)约束。参与即表示您应当遵守该准则。请将不可接受的行为报告至 [quack@duckdb.org](mailto:quack@duckdb.org)。

## **您发现了缺陷（bug）吗？**

* **请先在 GitHub 的 [Issues](https://github.com/duckdb/duckdb/issues) 中搜索，确认该缺陷尚未被报告。**
* 如果您找不到针对该问题的未关闭 issue，请[新建一个](https://github.com/duckdb/duckdb/issues/new/choose)。请务必包含**标题和清晰的描述**、尽可能多的相关信息，以及一个**代码示例**或**可执行的测试用例**来演示未按预期发生的行为。

## **您编写了修复缺陷的补丁吗？**

* 太好了！
* 如有可能，请添加一个单元测试用例，确保该问题不会再次出现。
* 请确保运行了代码格式化工具（`make format-fix`）。
* 在 GitHub 上打开一个新的 pull request 并附上补丁。
* 请确保 PR 描述清楚地说明了问题和解决方案。如适用，请包含相关的 issue 编号。

## 外部贡献者

* 在 GitHub 上与核心团队讨论您打算进行的更改
* 声明您正在或想要处理某个特定 issue
* 避免大型 pull request——它们被合并的可能性要小得多，因为审查起来极其困难

## Pull Request

* 不要直接提交/推送到 main 分支。请创建一个 fork 并提交 pull request。
* 维护分支时，请频繁与 main 合并。
* 维护分支时，请频繁向 main 提交 pull request。
* 如果您正在处理一个较大的 issue，请尽量将其拆分为若干较小的 issue。
* 请不要打开"Draft"状态的 pull request，并确保在打开 PR 之前先在您的 fork + 分支上运行 CI（见下文 CI 章节）。GitHub Actions 对开源项目的公开 fork 是免费的。
* 我们保留是否合并 pull request 的完全和最终决定权。遵守这些指南并不完全保证您的 pull request 会被合并。
* 请不要为修复小拼写错误而打开 pull request。

## Pull Request 的 CI

* Pull request 在合并前需要通过所有持续集成检查。
* 当本地 CI 的所有检查都正常后，您可以打开 PR，在 DuckDB 主仓库上运行 CI。向已打开的 pull request 提交更改会使其进入"Draft"状态。在这种情况下，当您应用了所有修复并在您的 fork 中通过后，可以将其标记为"Ready for Review"，以便再次运行公共 CI（通过 Web UI 右下角的按钮点击"ready for review"）。
* 请注意，CI 失败偶尔可能无关。您应该检查它是否与您的更改有关（因为如果是，则意味着您的更改破坏了某些东西）。否则，您应该 1）记住频繁与 main 合并并运行 make format-fix（有时需要运行 generate-files）2）检查其他 PR 的 CI 是否在同一测试上失败（这通常说明问题是由 CI 引起的临时问题）并且一般来说 3）调查您的更改与失败的 CI 之间没有重叠。

## 夜间 CI

* 软件包创建和长时间运行的测试将在夜间运行中执行
* 在您的 fork 上，您可以按照 https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow 中的信息为任意分支触发长时间运行的测试（NightlyTests.yml）

## 构建

* 要构建项目，请运行 `make`。
* 要构建用于调试的项目，请运行 `make debug`。
* 对于并行构建，您可以使用 [Ninja](https://ninja-build.org/) 构建系统：`GEN=ninja make`。
  * 默认的并行进程数可能会因 CPU 与内存的比例不同而锁死系统。如果发生这种情况，请限制最大构建进程数：`CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make`。
  * 不使用 Ninja 时，也可以通过设置 `CMAKE_BUILD_PARALLEL_LEVEL=$(nproc)` 来缩短构建时间。
* 为了加快重新构建，请安装 [ccache](https://ccache.dev/)。构建系统会自动检测并在可用时使用它。

## 测试

* 单元测试可以使用 sqllogictest 框架（`.test` 文件）编写，也可以直接用 C++ 编写。我们**强烈**倾向于使用 sqllogictest 框架编写测试。只有在绝对必要时（例如测试并发连接或其他特殊行为）才用 C++ 编写测试。
* 测试框架的文档可以在[此处](https://duckdb.org/dev/testing)找到。
* 多写测试。
* 使用不同的类型进行测试，尤其是数值、字符串和复杂的嵌套类型。
* 尝试测试意外/不正确的用法，而不仅仅是正常路径。
* `make unit` 运行**快速**单元测试（约一分钟），`make allunit` 运行**所有**单元测试（约一小时）。
* 在发送 PR 之前，请确保**所有**单元测试通过。
* 较慢的测试应添加到**所有**单元测试中。您可以通过在 sqllogictest 中将测试文件命名为 `.test_slow`，或在 C++ 测试中将测试组后添加 `[.]` 来实现。
* 查看您分支的代码覆盖率报告，并尝试在快速单元测试中覆盖所有代码路径。也尝试触发异常。允许部分异常未被触发（例如内存不足异常或类型切换异常），但大型代码分支应始终被覆盖或删除。
* DuckDB 使用 GitHub Actions 作为其持续集成（CI）工具。您也可以选择在您的 fork 仓库上运行 GitHub Actions。详细说明请参阅 [GitHub 文档](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)。在运行 GitHub Actions 之前，请确保您拥有 duckdb/duckdb 仓库的所有 Git 标签。为此，请执行以下命令 `git fetch <your-duckdb/duckdb-repo-remote-name> --tags`，然后
`git push --tags`。这些命令将从 duckdb/duckdb 仓库获取所有 git 标签并推送到您的 fork 仓库。这确保您拥有 GitHub Actions 工作流所需的全部标签。

## 格式化

* 使用制表符缩进，使用空格对齐。
* 行宽不应超过 120 列。
* 为确保格式一致，请使用 11.0.1 版本，可通过 `python3 -m pip install clang-format==11.0.1` 或 `pipx install clang-format==11.0.1` 安装。
* `clang_format` 和 `black` 会自动执行这些规则，使用 `make format-fix` 运行格式化工具。
* 项目还附带一个与这些规则对应的 [`.editorconfig` 文件](https://editorconfig.org/)。

## C++ 指南

* 不要使用 `malloc`，优先使用智能指针。关键字 `new` 和 `delete` 是代码坏味道。
* 强烈优先使用 `unique_ptr` 而非 `shared_ptr`，只有在**绝对**必要时才使用 `shared_ptr`。
* 尽可能使用 `const`。
* **不要**导入命名空间（例如 `using std`）。
* 核心代码（`src` 目录）中源文件的所有函数都应属于 `duckdb` 命名空间。
* 重写虚方法时，避免重复 virtual，并始终使用 `override` 或 `final`。
* 使用 `[u]int(8|16|32|64)_t` 而不是 `int`、`long`、`uint` 等。对于任何类型的偏移/索引/计数，使用 `idx_t` 而不是 `size_t`。
* 作为参数时优先使用引用而非指针。
* 对于非平凡对象的参数使用 `const` 引用（例如 `std::vector` 等）。
* 尽可能使用 C++11 的 for 循环：`for (const auto& item : items) {...}`
* 为 `if` 语句和循环使用花括号。避免单行 if 语句和循环，尤其是嵌套的。
* **类布局：** 以一个包含构造函数和公有变量的 `public` 块开始，接着是一个包含类公有方法的 `public` 块。之后是任何私有函数和私有变量。例如：
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
* 避免[未命名的魔法数字](https://en.wikipedia.org/wiki/Magic_number_(programming))。而是使用存储在 `constexpr` 中的命名变量。
* [尽早返回](https://medium.com/swlh/return-early-pattern-3d18a41bba8)。避免深层嵌套分支。
* 不要在 pull request 中包含被注释掉的代码块。
* 始终使用 `enum class`，绝不使用 C 风格的 `enum`。仅当枚举需要被序列化时，才指定显式存储布局（`: uint8_t`）。

## 错误处理

* **仅**在遇到终止查询的错误时（例如解析错误、表未找到）使用异常。异常应仅用于**异常**情况。对于不破坏执行流程的常规错误（例如您**预期**可能发生的错误），请改用返回值。
* 尝试添加触发异常的测试用例。如果异常无法通过测试用例轻松触发，那么它可能应该是断言。这并非总是成立（例如内存不足错误是异常，但很难触发）。
* 使用 `D_ASSERT` 进行断言。仅当断言失败意味着程序员错误时才使用**断言**。断言绝不应由用户输入触发。避免没有注释或上下文的代码，如 `D_ASSERT(a > b + 3);`。
* 大量使用断言，但要在断言旁的注释中清楚说明断言触发时出了什么问题。

## 命名约定

* 选择描述性的名称。避免单字母变量名。
* 文件：小写并用下划线分隔，例如 abstract_operator.cpp
* 类型（类、结构体、枚举、typedef、using）：以大写字母开头的驼峰式，例如 BaseColumn
* 变量：小写并用下划线分隔，例如 chunk_size
* 函数：以大写字母开头的驼峰式，例如 GetChunk
* 在**嵌套**循环中避免使用 `i`、`j` 等。优先使用例如 **column_idx**、**check_idx**。在**非嵌套**循环中可以使用 **i** 作为迭代索引。
* 这些规则部分由 `clang-tidy` 强制执行。

## 生成式 AI 政策

请不要提交由 AI（LLM）生成的 pull request。
审查此类 PR 会给维护者带来相当大的负担。
