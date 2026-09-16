# f2py-core（f2py-core）

> 本文是 `f2py` 域下的叶子子系统文档。域级总览见 `../f2py.md`。
> 本文只展开 f2py 的"Fortran 解析 → C 包装代码生成 → 运行时"管道，不重复展开 NumPy 本体其他域。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 命令行入口 | `f2py` CLI 与 Python API `run_main` | `numpy/f2py/f2py2e.py:426/751` |
| 输入解析 | 解析命令行得到文件列表与选项 | `f2py2e.py:195`（`scaninputline`） |
| Fortran 语法解析 | 自研 Fortran 解析器，产出 AST 字典 | `numpy/f2py/crackfortran.py` |
| 类型/签名映射 | Fortran 类型 → C/Python 签名表 | `numpy/f2py/capi_maps.py`、`rules.py`、`cb_rules.py`、`common_rules.py` |
| C 包装生成 | 为每个 Fortran 子程序生成 C 包装与 Python 模块 | `f2py2e.py:381`（`buildmodules`）、`cfuncs.py` |
| F90 模块/use 处理 | 处理 Fortran 90 module 与 use 语句 | `f90mod_rules.py`、`use_rules.py` |
| 回调/C 接口 | 回调函数与 ISO C Binding 映射 | `cb_rules.py`、`_isocbind.py`、`_isofenv.py` |
| 运行时 C 对象 | fortranobject.c/h：Fortran 数组→numpy 数组的 C 层桥 | `numpy/f2py/src/fortranobject.c`、`fortranobject.h` |
| 编译编排 | `-c` 模式调用系统编译器构建扩展 | `f2py2e.py:592`（`run_compile`） |

对外暴露点：命令行 `f2py`、Python `numpy.f2py.run_main`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `run_main(comline_list)` | `f2py2e.py:426` | 主流程：解析→crackfortran→buildmodules，返回模块依赖字典 |
| `callcrackfortran(files, options)` | `f2py2e.py:338` | 装配 crackfortran 全局选项并调用解析 |
| `crackfortran(files)` | `crackfortran.py` | 顶层解析函数，返回 postlist（AST 字典列表） |
| `buildmodules(lst)` | `f2py2e.py:381` | 调 `cfuncs.buildcfuncs()` 生成 C 包装与 Python 模块骨架 |
| `analyzeline / crackline` | `crackfortran.py:989/697` | 逐行识别 Fortran 声明/语句 |
| `postcrack/postcrack2` | `crackfortran.py:2037/2005` | 后处理 AST（隐式规则、common 块） |
| `sign2map / routsign2map / modsign2map` | `capi_maps.py` | 把子程序签名映射为 C 包装签名表 |
| `fortranobject.c` | `src/fortranobject.c` | 运行时：`f2py_vec_*` 等把 Fortran 数组指针包装为 ndarray |

## 3. 关键调用链

**链路：`f2py -m foo bar.f` 生成并编译扩展**（与 `f2py-core-dataflow.html` 对应）
1. `main()`（`f2py2e.py:751`）→ `run_main(sys.argv[1:])`（`:426`）。
2. `scaninputline`（`:195`）把命令行拆成 `files` 与 `options`；`capi_maps.load_f2cmap_file` 加载类型映射。
3. `callcrackfortran`（`:338`）把选项注入 `crackfortran` 全局，调 `crackfortran.crackfortran(files)` 产出 `postlist`（每个块一个 AST 字典）。
4. `buildmodules(postlist)`（`:381`）→ `cfuncs.buildcfuncs()` 遍历 AST，按 `capi_maps`/`rules` 模板生成 `<modulename>module.c` 与 `-f2pywrappers.f`。
5. 返回字典中每个模块标注 `csrc/h = fortranobject.c/.h`（`:510`），编译时一并链接。
6. `run_compile`（`:592`）再调用系统 C/Fortran 编译器把上述 C/Fortran 源编译为 `.so`。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|------|------|------|
| `-m MODULE` | 生成的 Python 模块名 | `f2py2e.py` |
| `-h <file.pyf>` | 仅导出签名文件（停止代码生成） | `f2py2e.py:460` |
| `--only/--skip` | 只包装/跳过指定子程序 | `crackfortran.py` |
| `include_paths` | 查找 Fortran include 的目录 | `f2py2e.py:348` |
| `do-lower` | 是否把 Fortran 名转小写 | `f2py2e.py:349` |
| `requires_gil` | 生成的包装是否持有 GIL（`Py_MOD_GIL_USED/NOT_USED`） | `f2py2e.py:372` |
| `f2cmap_file` | 自定义 Fortran 类型映射文件 | `capi_maps.py` |

## 5. 错误与重试语义

- **Fortran 语法不识别**：`crackfortran` 输出诊断（`outmess`），解析失败则不生成代码。
- **模块名冲突**：`validate_modulename`（`f2py2e.py:737`）在 pyf 已定义模块名时报错。
- **非 python module 块**：`run_main` 末尾校验所有块为 python module，否则 `TypeError`（`:501`）。
- **编译失败**：`run_compile` 透传编译器退出码；本层不重试。
- 无退避/重试；错误即终止生成。

## 6. 并发细节

- 代码生成是单线程 Python 文本处理，GIL 内执行。
- **生成的 C 扩展 GIL 行为**：`requires_gil` 控制包装是否释放 GIL；默认 `Py_MOD_GIL_NOT_USED` 以允许 numpy 数组运算并行。
- `fortranobject.c` 运行时在调用 Fortran 子程序期间按上述标志持有/释放 GIL；多线程调用需用户自行保证 Fortran 线程安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/f2py/`：`f2py2e.py`、`crackfortran.py`、`capi_maps.py`、`rules*.py`、`cfuncs.py`、`auxfuncs.py`、`src/fortranobject.c/.h`。

**Out-of-Scope（不在本仓库源码内）**
- 系统 C/Fortran 编译器（gcc/ifort 等）——外部工具。
- meson 构建系统本身——见 `build-meson` 叶子。
- 被包装的用户 Fortran 源码——外部输入。

## 8. 与相邻子系统交互

- **上游**：用户命令行；`numpy/_build_utils`/meson 在构建 NumPy 自身时调用 f2py 生成部分扩展。
- **本叶子 → 下游**：生成的 C 包装依赖 `fortranobject.c` 运行时与 Python C-API、NumPy C-API（`_core`）。
- **相邻**：与 `build-meson`（构建系统）、`code-gen`（另一套 .src 模板代码生成）同属"构建期代码生成"能力缝，但 f2py 面向 Fortran 接口。

## 9. 语言专项适配口径（纯 Python + 代码生成）

- **代码生成管道（重点）**：f2py 是"Fortran 文本 → AST 字典 → 模板 → C/Fortran 源"的完整代码生成器；`crackfortran` 是自研前端，`capi_maps`/`rules` 是映射表，`cfuncs` 是后端代码生成。
- **"生成 vs 手写"边界**：`fortranobject.c/.h` 是手写运行时；`*module.c`、`*-f2pywrappers.f` 是每次构建生成物，不入库、禁止手改。
- **注册表/映射**：`capi_maps.sign2map` 等是"类型签名字符串 → C 包装模板"的静态表，属配置驱动分派。
- **可选依赖**：f2py 运行需系统 Fortran 编译器，缺失时相关扩展构建跳过。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `f2py-core-architecture.html` | architecture | standard |
| 代码生成管道图 | `f2py-core-dataflow.html` | dataflow | standard |

- 降档原因：architecture 中 rules→gen 竖边标签与节点重叠，`labelDy` 补偿后仍落 standard；dataflow 受"同阶段竖边标签自动落点偏近源节点"约束，去重节点后落 standard。两图 render 退出码均为 0、HTML 约 800KB。
- 未补时序图：代码生成是线性管道（dataflow 已表达阶段流），时序图增量价值低。
- JSON IR 源文件：`json/f2py-core-architecture.json`、`json/f2py-core-dataflow.json`。
