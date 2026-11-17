## 通过 URL 提供扩展

执行 `INSTALL name` 时，如果文件本地不存在或没有通过静态链接打包，它将从网络下载。
扩展通过如下 URL 提供：http://extensions.duckdb.org/v0.8.1/windows_arm64/name.duckdb_extension.gz

拆解如下：
```
http://extensions.duckdb.org/              扩展注册表 URL（可包含子文件夹）
v0.8.1/                                     版本标识符
osx_arm64/                                  此扩展兼容的平台
name                                        给定扩展的默认名称
.duckdb_extension                           固定文件标识符
.gz                                         扩展以 gzip 文件形式提供
```

扩展注册表默认为 DuckDB 的注册表，但可以通过 `SET custom_extension_repository="http://some.url.org/subfolder1/subfolder2/"` 设置。

## 本地扩展

`INSTALL name` 将下载、解压并将扩展本地保存到类似 `~/.duckdb/extensions/v0.8.1/osx_arm64/name.duckdb_extension` 的路径。

```
~/.duckdb/                                  本地配置文件夹
extensions/                                 专用于扩展的固定子文件夹
v0.8.1/                                     版本标识符
osx_arm64/                                  此扩展兼容的平台
name                                        给定扩展的默认名称
.duckdb_extension                           固定文件标识符
```

配置文件夹默认位于主目录，但可以通过 `SET home_directory='/some/folder'` 覆盖。

## WebAssembly 可加载扩展（仍在变动）

```
https://extensions.duckdb.org/ 	            扩展注册表 URL（可包含子文件夹）
wasm/                                       wasm 子文件夹，必需
v1.27.0/                                    DuckDB-Wasm 版本标识符
webassembly_eh/                             此扩展兼容的平台/特性集
name                                        给定扩展的默认名称
.duckdb_extension                           固定文件标识符
```

DuckDB-Wasm 扩展由浏览器下载，且不会附加 .gz，因为解压缩状态通过 `Accept-Encoding: *` 和 `Content-Encoding: br` 等头部协商。

### 版本标识符

可以是 git 标签（`v0.8.0`、`v0.8.1` 等）或给定 duckdb 版本的 git 哈希。
它在编译期选择并固化到 DuckDB 中。给定的 duckdb 可执行文件或库绑定到单个版本标识符。

### 平台

在编译期通过平台检测固定并固化到 DuckDB 中。

### 扩展名称

扩展名称应以字母开头，仅使用 ASCII 小写字母、数字、点（'.'）或下划线（'_'），并具有合理的长度（< 64 个字符）。
