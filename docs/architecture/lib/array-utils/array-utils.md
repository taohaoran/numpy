# array-utils（array-utils）

> 本文是 `lib` 域下的叶子子系统文档。域级总览见 `../lib.md`。
> 本文只展开 NumPy 中"公共数组辅助函数族"，不重复展开 `npyio`（磁盘 IO）与 `stride-tricks`（视图技巧）。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

本叶子是 `numpy/lib` 下最杂的工具集合，按能力缝归为六个子模块（实现均在 `_*_impl.py`，薄包装保留原 `numpy/lib/xxx.py` 路径）。

| 能力缝 | 说明 | 源码路径 |
|------|------|----------|
| 集合运算 | unique / intersect1d / union1d / setdiff1d / setxor1d / isin / ediff1d | `numpy/lib/_arraysetops_impl.py` |
| 统计与数值 | median / percentile / quantile / average / cov / corrcoef / histogram2d / gradient / diff / interp | `numpy/lib/_function_base_impl.py` |
| vectorize | 解析 gufunc 签名并逐元素广播的"伪 ufunc" | `numpy/lib/_function_base_impl.py:2308`（`class vectorize`） |
| NaN 感知归约 | nanmin/nanmax/nansum/nanmean/nanmedian/nanvar/nanstd/nanpercentile | `numpy/lib/_nanfunctions_impl.py` |
| 形状与二维 | split/array_split/expand_dims/column_stack/tile/kron；tri/tril/triu/eye/diag/vander/flip | `_shape_base_impl.py`、`_twodim_base_impl.py` |
| 类型与复数 | real/imag/iscomplex/nan_to_num/common_type/mintypecode | `numpy/lib/_type_check_impl.py` |
| 复数域 scimath | 支持负数开方等复数数学的 sqrt/log/power/arccos | `numpy/lib/_scimath_impl.py` |
| 运算符 Mixin | 把 Python 算术特例方法转发到 `__array_ufunc__` | `numpy/lib/mixins.py:60`（`NDArrayOperatorsMixin`） |
| 通用工具 | _ureduce 归约分发、normalize_axis_* 等 | `_utils_impl.py`、`_type_check_impl.py` |

对外暴露点：经 `numpy/__init__.py` 导出为 `np.unique`、`np.median`、`np.vectorize` 等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `_xxx_dispatcher` 系列 | 各 `_*_impl.py` 紧邻公开函数 | `__array_function__` 协议的分派器，声明哪些参数走协议重载 |
| `unique` / `_unique1d` | `_arraysetops_impl.py:146/350` | 一维去重；`unique_all/counts/inverse/values` 为 NamedTuple 结果族 |
| `isin` / `_isin` | `_arraysetops_impl.py:959/806` | 元素是否在测试集合中；`kind` 选排序/广播算法 |
| `vectorize` | `_function_base_impl.py:2308` | 类，封装标量函数为按元素广播的"函数"；解析 gufunc 签名 |
| `_parse_gufunc_signature` | `_function_base_impl.py:2189` | 解析 `(m,n)->(p)` 形式的 gufunc 签名 |
| `_ureduce` | `_function_base_impl.py:3857` | 统一 axis/keepdims 归约的辅助分发 |
| `percentile/quantile` 族 | `_function_base_impl.py:4094/4296` | 含 `_compute_virtual_index`/`_get_gamma` 多插值方法 |
| `_replace_nan / _nan_mask` | `_nanfunctions_impl.py:70/43` | 把 NaN 替换为可归约值或掩码后再调用普通归约 |
| `apply_along_axis` | `_shape_base_impl.py:277` | 沿某轴把 1D 切片交给用户函数 |
| `NDArrayOperatorsMixin` | `mixins.py:60` | 用 `_binary_method/_numeric_methods` 工厂批量生成 `__add__` 等特例方法 |
| `_binary_method` 等闭包工厂 | `mixins.py:17-52` | 把一个 ufunc 闭包成比较/算术特例方法 |

## 3. 关键调用链

**链路一：`np.unique` 去重**
1. `unique(ar, ...)`（`_arraysetops_impl.py:146`）先展平为 1D。
2. 调 `_unique1d`（`:350`）：若 `not assume_unique`，先用排序 ufunc 排序，再识别连续相等段。
3. 按 `return_index/return_inverse/return_counts` 用布尔/索引运算回算下标，返回 `Unique*Result` NamedTuple。

**链路二：`np.nanmean`**
1. `nanmean`（`_nanfunctions_impl.py:958`）调 `_replace_nan(a, 0)`（`:70`）把 NaN 位置置 0 并返回掩码计数。
2. 对掩码后数组调 `np.sum` 类归约，再用 `_divide_by_count`（`:204`）除以非 NaN 计数，避免 0/0。

**链路三：`np.vectorize(func)` 调用**
1. 构造时 `_parse_gufunc_signature`（`_function_base_impl.py:2189`）解析签名，`_parse_input_dimensions`（`:2248`）对齐核心维。
2. 调用时广播输入、逐元素把标量传给 `func`，用 `_get_vectorize_dtype`（`:2301`）推断输出 dtype。

**链路四：`NDArrayOperatorsMixin` 子类相加**
1. 子类实例 `x + y` 触发 `__add__ = _numeric_methods(um.add,'add')` 生成的方法（`mixins.py:153`）。
2. 该方法调用 `self.__array_ufunc__(np.add, '__call__', self, other)`，由子类实现决定如何解包/重包装。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `assume_unique` | unique/isin/intersect1d 等若输入已去重则跳过排序 | `_arraysetops_impl.py:146/667` |
| `kind`（isin） | 选排序法 `'sort'` 或广播法 `'searchsorted'`/`'table'` | `_arraysetops_impl.py:959` |
| `method`（quantile） | 线性/低/高/中点/最近/插值等 9 种插值 | `_function_base_impl.py:4296` |
| `overwrite_input` | 归约是否可覆盖输入数组 | `_function_base_impl.py:3945` |
| `_HANDLED_TYPES` | mixin 子类声明可与哪些类型交互 | `mixins.py:84`（示例） |
| `like`（`np=True` 协议） | 支持 array-like 协议的关键字 | `_twodim_base_impl.py:180`（eye 等） |

## 5. 错误与重试语义

- **空数组全 NaN**：`nanmean/nanvar` 等对全 NaN 切片抛 `RuntimeWarning` 并返回 NaN（不报错中断整体）。
- **quantile 越界**：`_quantile_is_valid`（`_function_base_impl.py:4561`）检查 q∈[0,1]，越界 `ValueError`。
- **vectorize 签名**：gufunc 签名解析失败抛 `ValueError`（`_parse_gufunc_signature`）。
- **iscomplex/isreal 等**：纯判断函数，不抛错，返回布尔。
- 本叶子为纯计算工具，无重试/退避；参数校验失败即抛。

## 6. 并发细节

- 纯 Python 编排层，受 GIL 约束；实际排序/归约/比较下沉到 `_core` 的 C/UMath ufunc 执行（释放 GIL，多线程可并行）。
- Python 层本身无锁、无后台线程；`apply_along_axis` 逐轴切片顺序调用用户函数，单线程。
- `vectorize` 是显式的 Python 级逐元素循环，性能远低于真正 ufunc（设计如此，仅为接口兼容）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/lib/_arraysetops_impl.py`、`_function_base_impl.py`、`_nanfunctions_impl.py`、`_shape_base_impl.py`、`_twodim_base_impl.py`、`_type_check_impl.py`、`_scimath_impl.py`、`_polynomial_impl.py`、`_utils_impl.py`、`mixins.py`。

**Out-of-Scope（不在本仓库源码内）**
- 排序、归约、广播的底层 ufunc 实现位于 `_core`（其他域，C 层）。
- `numpy._NoValue` 哨兵与 `set_module` 装饰器来自 `_utils`。
- 掩码数组的 NaN/掩码语义由 `numpy/ma`（`ma-polynomial-matrix` 域）补充；本叶子的 nanfunctions 只处理 NaN 而非掩码。

## 8. 与相邻子系统交互

- **上游**：`numpy/__init__.py` 再导出；`__array_function__` dispatcher 让第三方数组库（如 Dask、xarray）能拦截这些函数。
- **本叶子 → 下游**：大量调用 `_core` 的 ndarray 方法与 ufunc（排序、归约、比较、广播）。
- **下游/相邻**：`mixins.py` 供 `numpy.ma.MaskedArray`、`numpy.matrix` 等子类复用运算符特例方法。
- 方向：用户 → array-utils 编排 → `_core` ufunc → 返回新 ndarray。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝归组**：本叶子不是 controller 模式，而是"按计算能力缝分组的纯 Python 薄编排层"——集合运算缝、统计归约缝、NaN 感知缝、形状变换缝、类型判定缝、运算符 mixin 缝。
- **协议缝（`__array_function__`）**：每个公开函数成对出现 `公开函数 + _xxx_dispatcher`，dispatcher 声明哪些参数参与多分派，是 NumPy 对外的扩展点能力缝。
- **闭包工厂**：`mixins.py` 的 `_binary_method/_numeric_methods/_unary_method` 是元编程式工厂，把 ufunc 批量转成 Python 特例方法，避免手写 30+ 个方法（DRY）。
- **双轨实现**：统计归约有"带 NaN 版本"（nanfunctions，先掩码再归约）与"普通版本"（_core 直接 ufunc），运行期按是否含 NaN 选择路径。
- **代码生成/懒加载**：无；薄包装 `array_utils.py` 等是 `_impl` 拆分后的再导出接缝。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `array-utils-architecture.html` | architecture | standard |

- 降档原因：本叶子子模块多（六个能力缝 + 下游计算层），中间列竖向堆叠的子模块与下游连接在 showcase 严格布局下触发约束，降 standard 渲染；render 退出码 0、HTML 约 809KB。
- 未补时序/数据流图：本叶子是薄编排层，核心逻辑在 `_core` ufunc，Python 侧调用链用第 3 节文字化已足够，补图增量价值低，故仅 1 张架构图。
- JSON IR 源文件：`json/array-utils-architecture.json`。
