# code-gen（code-gen）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 `../support.md`。
> 本文只展开 NumPy 构建期的 C 代码生成机制，不重复展开其他支撑子系统。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| .src 模板渲染 | 把 `.c.src`/`.h.src`（Tempita 语法）渲染为 `.c`/`.h` | `numpy/_build_utils/process_src_template.py:22` |
| 模板处理器 | 加载 vendored tempita 的 `conv_template.process_file` | `numpy/_build_utils/conv_template.py` |
| ufunc 代码生成 | 由 Ufunc/TypeDescription 表生成 umath 注册表与循环分发 C 代码 | `numpy/_core/code_generators/generate_umath.py:1559/1655/1690` |
| ufunc docstring 生成 | 生成 ufunc 文档字符串 | `numpy/_core/code_generators/ufunc_docstrings.py` |
| C-API 导出表生成 | 扫描 C 头文件 `API` 标注，生成 numpy_api / ufunc_api 导出表 | `numpy/_core/code_generators/genapi.py`、`generate_numpy_api.py`、`generate_ufunc_api.py` |
| API 版本校验 | 校验 C-API 版本号一致性 | `numpy/_core/code_generators/verify_c_api_version.py`、`cversions.txt` |

代表性 `.src` 模板：`numpy/_core/src/multiarray/arraytypes.c.src`、`scalartypes.c.src`、`lowlevel_strided_loops.c.src` 等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `process_and_write_file(fromfile, outfile)` | `process_src_template.py:22` | 渲染一个 `.src` 模板并写出 |
| `conv_template.process_file` | `_build_utils/conv_template.py` | Tempita 模板实例化 |
| `Ufunc` / `TypeDescription` / `TD` | `generate_umath.py:193/42/154` | 描述一个 ufunc 及其各类型签名 |
| `make_ufuncs / make_code` | `generate_umath.py:1559/1655` | 由定义表生成 C 代码 |
| `Function` / `TypeApi` / `FunctionApi` | `genapi.py:140/330/401` | 解析 C 头文件 API 标注、构造导出表 |
| `find_functions(filename, tag='API')` | `genapi.py:224` | 从头文件抽取 API 函数声明 |

## 3. 关键调用链

**链路一：.src 模板渲染**
1. meson 构建调用 `process_src_template.py infile.c.src -o outfile.c`（`main`，`process_src_template.py:36`）。
2. `get_processor()`（`:7`）用 `importlib` 动态加载 `conv_template.py`（构建期 numpy 未安装，不能直接 import）。
3. `process_file` 按 Tempita 语法实例化，写出 `.c`。

**链路二：generate_umath 生成 ufunc**
1. `main()`（`generate_umath.py:1690`）取内置 `defdict`（一组 `Ufunc`）。
2. `make_code(defdict, filename)`（`:1655`）→ `make_ufuncs`（`:1559`）→ `make_arrays`（`:1459`）生成注册表/循环 C 代码。
3. 写出 `__umath_generated.c`（或 `-o` 指定文件）。

**链路三：genapi 生成 C-API 导出表**
1. `find_functions`（`genapi.py:224`）扫描 C 头文件中带 `API` 标注的函数。
2. `FunctionApi`/`TypeApi` 收集后 `merge_api_dicts`、`get_api_functions` 生成导出表 C 源。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `.src` 扩展名约定 | 输入必须 `.c.src`/`.h.src`，否则 `ValueError` | `process_src_template.py:58` |
| `-o outfile` | 生成文件输出路径 | 各生成器 `main` |
| `cversions.txt` | C-API 版本号清单 | `code_generators/cversions.txt` |
| Tempita 变量 | 模板内 `{{...}}` 占位由构建变量填充 | `.src` 模板 |

## 5. 错误与重试语义

- **扩展名不符**：`process_src_template` 显式 `ValueError`（`:59`）。
- **API 版本不一致**：`verify_c_api_version` 在构建期校验，不符则构建失败。
- 无重试；构建期错误即终止。

## 6. 并发细节

- 构建期单线程代码生成（Python），GIL 内。
- 生成的 C 代码经 meson 可并行编译（外部并行）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/_core/code_generators/`、`numpy/_build_utils/process_src_template.py`、`conv_template.py`。

**Out-of-Scope（不在本仓库源码内）**
- vendored tempita（`numpy/_build_utils/tempita*`、`tempita/`）——第三方 vendored，不在本项目原创源码内。
- 生成的 `.c`/`.h` 产物——不入库，构建时产生。
- C 编译器与 meson——外部。

## 8. 与相邻子系统交互

- **上游**：`build-meson`（meson.build）在构建期调用这些生成器。
- **本叶子 → 下游**：生成 C 源交给 `_core` 的 C 编译；`.src` 模板散布在 `numpy/_core/src/**`。
- 与 f2py 同属"构建期代码生成"，但 code-gen 面向 C 内部源，f2py 面向 Fortran 接口。

## 9. 语言专项适配口径（纯 Python 项目专项 / 代码生成重点）

- **"生成 vs 手写"边界（重点）**：`.src` 模板与 `generate_umath`/`genapi` 是手写源，构建期产物 `.c`/`.h` 不入库、禁止手改；这是 NumPy 避免 C 层重复的核心 DRY 机制。
- **构建期不可 import numpy**：`process_src_template.get_processor` 用 `importlib.util` 动态加载，因为 numpy 自身尚未构建完成——典型的自举（bootstrap）约束。
- **代码生成器**：generate_umath 用类/数据表（`Ufunc`/`TD`）描述 ufunc，是"数据驱动代码生成"；genapi 用 AST 轻量解析头文件标注。
- **vendored**：tempita 为第三方 vendored 模板引擎，标注外部。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `code-gen-architecture.html` | architecture | standard |
| 生成管道图 | `code-gen-dataflow.html` | dataflow | standard |

- 降档原因：初版 architecture 中间列节点导致 umath→c 穿线，去 genpyn 节点后落 standard；dataflow 因 4 行节点需加高 viewBox。两图 render 退出码均 0、HTML 约 800KB。
- JSON IR 源文件：`json/code-gen-architecture.json`、`json/code-gen-dataflow.json`。
