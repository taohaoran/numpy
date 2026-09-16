# testing-utils（testing-utils）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 `../support.md`。
> 本文只展开 NumPy 自带的测试工具库，不重复展开 `typing-stubs`/`code-gen`/`build-meson`。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 数值断言 | assert_equal/almost_equal/array_* /allclose/ulp | `numpy/testing/_private/utils.py` |
| 比较公共实现 | assert_array_compare：多数数组断言的核心 | `utils.py:759` |
| 异常/警告断言 | assert_raises/assert_warns/assert_no_warnings | `utils.py:1529/2021/2101` |
| 警告抑制夹具 | suppress_warnings / clear_and_catch_warnings | `utils.py:2312/2246` |
| 临时路径夹具 | tempdir / temppath | `utils.py:2210/2225` |
| 引用计数/GC 检查 | _assert_valid_refcount / assert_no_gc_cycles | `utils.py:1666/2653` |
| 即时编译扩展 | build_and_import_extension / compile_extension_module | `numpy/testing/_private/extbuild.py:18/83` |
| hypothesis 策略 | 生成测试用数组示例 | `numpy/testing/_private/hypothesis_helpers.py` |
| 测试装饰/度量 | decorate_methods / measure / rundocs | `utils.py:1574/1620/1447` |
| 公共 API 再导出 | `numpy/testing/__init__.py` 汇总 `_private.utils` | `numpy/testing/__init__.py:11-16` |

对外暴露点：`numpy.testing.assert_array_equal` 等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `assert_array_compare(comparison, x, y, ...)` | `utils.py:759` | 通用逐元素比较，生成错误信息；其他 assert_array_* 复用 |
| `assert_allclose` | `utils.py:1692` | 用 rtol/atol 做浮点近似比较 |
| `assert_array_almost_equal_nulp / max_ulp` | `utils.py:1806/1869` | 以单位末位（ULP）度量浮点差异 |
| `suppress_warnings` | `utils.py:2312` | 上下文管理器，记录/过滤警告 |
| `build_and_import_extension` | `extbuild.py:18` | 测试期把一段 C 源码编译为扩展并 import |
| `compile_extension_module` | `extbuild.py:83` | 调系统编译器编译 C 扩展 |
| `KnownFailureException` | `utils.py:52` | 标记已知失败的测试 |

## 3. 关键调用链

**链路：`assert_allclose(a, b, rtol, atol)`**
1. 委托 `assert_array_compare`（`utils.py:759`），传入"绝对差 ≤ atol + rtol*|b|"的比较函数。
2. 逐元素比较，若有不通过元素，`build_err_msg`（`:252`）生成差异报告。
3. 失败则 `AssertionError`，由 pytest 捕获展示。

**链路：测试中编译扩展**
1. `build_and_import_extension`（`extbuild.py:18`）调 `_make_source` 拼 C 源码、`_c_compile` 调编译器。
2. 产出 `.so` 后 import 进测试进程。

## 4. 配置项

| 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `rtol/atol`（assert_allclose） | 1e-7 / 0 | `utils.py:1692` |
| `decimal`（assert_almost_equal） | 7 位小数 | `utils.py:533` |
| `verbose` | 是否打印详细差异 | 多数断言 |
| `RUN_SLOW` 环境变量 | 控制慢速测试开关（测试套件级） | 测试配置 |

## 5. 错误与重试语义

- **断言失败**：抛 `AssertionError`，pytest 终止该用例；不重试。
- **扩展编译失败**：`extbuild` 透传编译器错误；测试可 skip。
- **警告未抑制**：`assert_no_warnings` 触发时报错。
- 无业务重试。

## 6. 并发细节

- 纯 Python 测试工具，GIL 下单线程；无后台线程。
- 断言比较本身可调 `_core` ufunc（释放 GIL）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/testing/`：`__init__.py`、`overrides.py`、`_private/utils.py`、`extbuild.py`、`hypothesis_helpers.py`。

**Out-of-Scope（不在本仓库源码内）**
- pytest / unittest / hypothesis 框架本体——外部。
- C 编译器——外部。

## 8. 与相邻子系统交互

- **上游**：NumPy 测试套件与下游库经 `numpy.testing` 写测试。
- **本叶子 → 下游**：调用 `_core` 数组做比较；`extbuild` 调用外部编译器。
- 与 `code-gen`/`build-meson` 同属构建/测试支撑。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝**：断言族是"比较策略可插拔"缝——`assert_array_compare` 接收一个比较函数，衍生出 equal/less/close 等一族断言。
- **夹具/上下文管理器**：suppress_warnings、tempdir 是测试领域的上下文管理器缝。
- **可选依赖**：hypothesis、C 编译器为可选，缺失时相关测试跳过。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `testing-utils-architecture.html` | architecture | standard |

- 降档原因：ext→hyp 竖边标签重叠，去边后落 standard；render 退出码 0、HTML 约 802KB。
- 未补时序/数据流图：测试工具多为独立函数，调用链短，补图增量价值低。
- JSON IR 源文件：`json/testing-utils-architecture.json`。
