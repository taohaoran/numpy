# 函数基与形状操作（function-base）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
> 本文展开 NumPy 的形状操作函数、数值序列生成函数、einsum 优化器及 C 对象文档注入机制。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 维度提升 | atleast_1d、atleast_2d、atleast_3d：将标量/低维输入提升到至少指定维度 | `numpy/_core/shape_base.py:20,79,138` |
| 轴堆叠 | stack（沿新轴堆叠）、vstack（垂直堆叠）、hstack（水平堆叠）、unstack（拆栈） | `numpy/_core/shape_base.py:377,219,293,470` |
| 块拼接 | block：嵌套列表/数组的递归块拼接，支持任意维度嵌套 | `numpy/_core/shape_base.py:777` |
| 等差序列 | linspace（线性等距）、logspace（对数等距）、geomspace（几何等距） | `numpy/_core/function_base.py:28,210,325` |
| einsum 优化 | einsum：爱因斯坦求和，支持 optimize='greedy'/'optimal' 路径优化；einsum_path：仅计算收缩路径 | `numpy/_core/einsumfunc.py:1243,635` |
| einsum 收缩路径搜索 | _optimal_path（暴力搜索最优收缩顺序）、_greedy_path（贪心近似）、_find_contraction（单步收缩） | `numpy/_core/einsumfunc.py:150,330,90` |
| C 对象文档注入 | add_newdoc：向 C 扩展对象的 __doc__ 注入文档字符串；_add_newdocs.py 在 import 时批量执行 | `numpy/_core/function_base.py:520`、`numpy/_core/_add_newdocs.py` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `atleast_1d/2d/3d(*arys)` | `shape_base.py:20,79,138` | 维度提升，标量变 1D，0D 变 2D 等 |
| `stack(arrays, axis=0, ...)` | `shape_base.py:377` | 沿新轴堆叠数组序列 |
| `block(arrays)` | `shape_base.py:777` | 嵌套结构块拼接，递归展开列表树 |
| `_block_info_recursion(arrays, max_depth, ...)` | `shape_base.py:693` | block 的递归信息计算，确定各块位置和形状 |
| `linspace(start, stop, num=50, ...)` | `function_base.py:28` | 线性等距采样，支持 endpoint/retstep/dtype/axis |
| `einsum(*operands, optimize=False, ...)` | `einsumfunc.py:1243` | 爱因斯坦求和表达式求值 |
| `einsum_path(*operands, optimize='greedy', ...)` | `einsumfunc.py:635` | 仅计算并返回最优收缩路径，不执行计算 |
| `_optimal_path(input_sets, output_set, idx_dict, memory_limit)` | `einsumfunc.py:150` | 暴力搜索最优收缩顺序（O(n!) 复杂度） |
| `_greedy_path(input_sets, output_set, idx_dict, memory_limit)` | `einsumfunc.py:330` | 贪心搜索：每步选 FLOPS 最小的收缩 |
| `add_newdoc(place, obj, doc, ...)` | `function_base.py:520` | 向指定 C 对象注入文档字符串 |

## 3. 关键调用链

### 3.1 np.einsum 调用链

1. 用户调用 `np.einsum('ij,jk->ik', a, b)` → 进入 `einsumfunc.py:1243` 的 `einsum()`
2. `_parse_einsum_input()` 解析 einsum 表达式，提取输入/输出下标集合和尺寸字典（`einsumfunc.py:445`）
3. 若 `optimize` 为 True/'greedy'/'optimal'：调用 `einsum_path()` 计算收缩顺序（`einsumfunc.py:635`）
4. `einsum_path` 内部调用 `_greedy_path` 或 `_optimal_path` 搜索最优收缩顺序
5. 最终调用 C 扩展 `c_einsum(operands, sublist)` 执行实际收缩计算（`einsumfunc.py:9`）
6. 特殊优化：`_parse_eq_to_batch_matmul` 识别可批量矩阵乘的表达式，走 `bmm_einsum` 专用路径（`einsumfunc.py:969,1144`）

### 3.2 np.block 调用链

1. 用户调用 `np.block([[a, b], [c, d]])` → 进入 `shape_base.py:777` 的 `block()`
2. `_block_setup(arrays)` 检测嵌套深度，确定最大维度（`shape_base.py:952`）
3. `_block_info_recursion(arrays, max_depth, result_ndim)` 递归计算各块位置和形状（`shape_base.py:693`）
4. `_block_slicing(arrays, list_ndim, result_ndim)` 按递归信息切片展开（`shape_base.py:967`）
5. `_block_concatenate` 沿各轴依次拼接（`shape_base.py:986`）

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `optimize`（einsum） | False（不优化，直接执行）；'greedy'（贪心近似）；'optimal'（暴力最优） | `einsumfunc.py:1243` |
| `num`（linspace/logspace/geomspace） | 50 个采样点 | `function_base.py:28,210,325` |
| `endpoint` | True（包含终点 stop） | 同上 |
| `memory_limit`（einsum_path） | 自动检测，限制中间张量最大元素数 | `einsumfunc.py:150,330` |
| `warn_on_python`（add_newdoc） | True，向纯 Python 对象添加文档时发警告 | `function_base.py:520` |

## 5. 错误与重试语义

- **einsum 表达式解析**：`_parse_einsum_input` 遇到重复下标、不一致维度时抛出 `ValueError`（`einsumfunc.py:445`）
- **block 深度不匹配**：`_block_check_depths_match` 验证嵌套列表深度一致，不一致时抛出 `ValueError`（`shape_base.py:555`）
- **add_newdoc 对 Python 对象**：若目标已是 Python 对象（非 C 扩展），`warn_on_python=True` 时发出警告（`function_base.py:489`）
- **linspace 整数 dtype 舍入**：NumPy 1.20 起整数 dtype 向 -inf 舍入（而非向 0），旧行为需手动 `.astype(np.int_)`（`function_base.py:39`）

## 6. 并发细节

- 本叶子为纯 Python 层，不管理线程/锁
- **einsum 路径搜索**为纯 CPU 计算（Python 层），不释放 GIL；实际张量收缩在 C 扩展 `c_einsum` 中执行，释放 GIL
- **block 递归**为单线程 Python 递归，无并发

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/shape_base.py`：维度提升、堆叠、块拼接
- `numpy/_core/function_base.py`：数值序列生成、add_newdoc
- `numpy/_core/einsumfunc.py`：einsum 表达式解析与路径优化
- `numpy/_core/_add_newdocs.py`：C 对象文档字符串批量注入

**Out-of-Scope（不在本仓库源码内）**

- `c_einsum` C 扩展实现（实际张量收缩计算）——在 `_multiarray_umath` C 扩展中
- `concatenate` C 扩展实现——在 `_multiarray_umath` 中
- 文档字符串内容——`_add_newdocs.py` 只是注入器，文档内容本身由 `add_newdoc` 调用参数提供

## 8. 与相邻子系统交互

- **上游**：用户代码通过 `np.stack`、`np.linspace`、`np.einsum` 等调用；`numpy/__init__.py` 从 `_core` 再导出
- **下游 → C 扩展**：`shape_base.py` → `multiarray.concatenate`；`einsumfunc.py` → `c_einsum`；`function_base.py` → `multiarray.asanyarray`
- **下游 → overrides**：所有公开函数经 `@array_function_dispatch` 包装
- **下游 → numeric**：`einsumfunc.py` 使用 `numeric.asanyarray`、`numeric.reshape`

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **能力缝归组**：核心能力缝是 **NEP-18 覆盖协议**（所有公开函数经 `@array_function_dispatch` 包装）和 **einsum 路径优化引擎**（纯 Python 层的收缩顺序搜索，将 NP 难的最优收缩问题在 Python 层求解，再将最优子列表传给 C 层执行）
2. **代码生成/文档注入**：`_add_newdocs.py` 是"运行时文档注入"机制——C 扩展对象在编译时无文档，Python 在 import 时通过 `add_newdoc` 批量注入文档字符串，避免每次修改文档都重新编译 C 扩展
3. **einsum 符号表**：`einsum_symbols` 硬编码 52 个字母（大小写），注释说明导入 `string.ascii_letters` 太慢（800µs），直接硬编码避免 import 开销（`einsumfunc.py:16-19`）

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| function-base 架构图 | `function-base-architecture.html` | architecture | standard |

- JSON IR 源文件：`json/function-base-architecture.json`
- 降档原因：standard 档通过渲染；showcase 校验因 addnewdocs→funcbase 边交叉 einsumfunc 组件而失败，移除该边后 standard 档通过
- 时序图/数据流图：einsum 调用链已在第 3 节文字化描述；einsum 路径搜索为线性 Python 算法，不适用额外时序图
