# 数值层（numeric-layer）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
> 本文展开 NumPy 纯 Python 前端的数值函数层与 C 扩展再导出薄层，不重复展开 `overrides.py` 的 NEP-18 协议细节（见 `../overrides-ufunc-config/`）、`numerictypes.py` 的类型系统（见 `../numerictypes-getlimits/`）。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 数组创建与操作函数 | zeros_like、ones、full、outer、tensordot、roll、moveaxis、cross、indices、fromfunction、isclose、allclose、array_equal、astype 等约 40 个 Python 级函数 | `numpy/_core/numeric.py` |
| 规约与排序函数 | sum、prod、mean、var、std、min、max、take、reshape、sort、argsort、argmax、nonzero、diagonal、trace、clip 等约 50 个函数 | `numpy/_core/fromnumeric.py` |
| C 扩展再导出 | 从 `_multiarray_umath` C 扩展 re-export ndarray、dtype、array、asarray 等核心对象；设置 `__module__` 兼容旧 pickle | `numpy/_core/multiarray.py` |
| require() 函数 | 将 array-like 转换为满足指定 flag 要求的 ndarray（C/F 连续、对齐、可写、拥有数据等） | `numpy/_core/_asarray.py` |
| ndarray 方法桥接 | 供 C 代码和 Python 命名空间函数共同调用的 reduce 桥接：_sum、_prod、_mean、_var、_std、_amin、_amax、_any、_all、_clip、_ptp | `numpy/_core/_methods.py` |
| dtype 内部工具 | 结构化 dtype 字段解析（_usefields、_makenames_list）、PEP 3118 格式串解析（_dtype_from_pep3118）、ctypes 桥（_ctypes 类）、view 安全检查（_view_is_safe）、结构化 dtype 提升（_promote_fields） | `numpy/_core/_internal.py` |
| 异常层级 | UFuncTypeError 基类及子类：_UFuncNoLoopError、_UFuncBinaryResolutionError、_UFuncCastingError、_ArrayMemoryError；_display_as_base 装饰器隐藏实现细节 | `numpy/_core/_exceptions.py` |
| 内存映射数组 | memmap 类（ndarray 子类），通过 mmap 模块实现磁盘文件内存映射，支持 r/r+/w+/c 模式 | `numpy/_core/memmap.py` |
| 向后兼容薄层 | `numpy/core/` 包经 `__getattr__` 惰性转发到 `_core` 并发出弃用警告 | `numpy/core/__init__.py` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `ndarray`（C 实现） | `_multiarray_umath` → `multiarray.py` re-export | NumPy 多维数组核心对象，C 扩展实现 |
| `dtype`（C 实现） | `_multiarray_umath` → `multiarray.py` re-export | 数据类型描述符 |
| `_wrapreduction(obj, ufunc, method, axis, dtype, out, ...)` | `fromnumeric.py:43` | 规约函数统一入口：若 obj 非纯 ndarray 则优先调用 obj 的规约方法，否则走 `ufunc.reduce` |
| `_wrapreduction_any_all(...)` | `fromnumeric.py:69` | any/all 的专用规约桥接，强制 dtype=bool |
| `require(a, dtype, requirements, *, like)` | `_asarray.py:24` | 将输入转换为满足指定 flag 要求的数组，支持 NEP-18 `like=` 协议 |
| `_NoValueType` / `_NoValue` | `numpy/_globals.py` | 单例哨兵值，区分"用户未传参"与"用户显式传 None" |
| `_display_as_base` | `_exceptions.py:16` | 装饰器将异常子类 `__name__` 改为基类名，隐藏实现细节 |
| `memmap(ndarray)` | `memmap.py:24` | 磁盘内存映射数组子类，`__array_priority__ = -100.0` |
| `_ctypes` | `_internal.py:261` | ndarray.ctypes 属性的 Python 包装，提供 data_as/shape_as/strides_as |
| `_ctypes.data_as(obj)` | `_internal.py:279` | 将数据指针转换为指定 ctypes 类型，保持数组引用防 GC |

## 3. 关键调用链

### 3.1 规约函数调用链（np.sum）

1. 用户调用 `np.sum(a, axis=0)` → 进入 `fromnumeric.py:2466` 的 `sum()` 函数（经 `@array_function_dispatch` 包装）
2. 内部调用 `_wrapreduction(a, um.add, 'sum', axis, dtype, out, keepdims=..., initial=..., where=...)`（`fromnumeric.py:43`）
3. `_wrapreduction` 检查 `type(obj) is not mu.ndarray`：若非纯 ndarray（如子类），优先调用 `obj.sum(axis=axis, ...)` 方法
4. 若 obj 是纯 ndarray 或方法调用失败，回退到 `ufunc.reduce(obj, axis, dtype, out, **passkwargs)`
5. `ufunc.reduce` 最终由 C 扩展 `_multiarray_umath` 的 `um.add.reduce` 实现

### 3.2 数组创建函数调用链（np.zeros_like）

1. 用户调用 `np.zeros_like(a)` → 进入 `numeric.py:98` 的 `zeros_like()`（经 `@array_function_dispatch(_zeros_like_dispatcher)` 包装）
2. dispatcher 返回 `(a,)` 供 NEP-18 协议检查是否有 `__array_function__` 覆盖
3. 若无覆盖，实际实现调用 `multiarray.zeros_like(a, dtype, order, subok, shape)` → C 扩展
4. 若有 `like=` 参数，走 `_require_with_like` 路径路由到第三方数组库

### 3.3 均值计算调用链（ndarray.mean → _methods._mean）

1. C 代码或 Python 命名空间调用 `_methods._mean(a, axis, dtype, ...)`（`_methods.py:115`）
2. `_count_reduce_items` 计算规约元素个数 `rcount`；空切片时发出 `RuntimeWarning`
3. dtype 推断：整数/bool 默认提升为 float64，float16 提升为 float32
4. 调用 `umr_sum(arr, axis, dtype, out, keepdims, where)` 求和
5. 结果除以 `rcount` 得到均值；float16 结果再转回原始 dtype

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `like` 参数（NEP-18） | None，不启用第三方数组库覆盖 | 所有 `@array_function_dispatch` 函数 |
| `requirements`（require 函数） | None，返回 `asanyarray(a)` | `_asarray.py:101` |
| `mode`（memmap） | `'r+'`（读写），支持 `'r'`/`'r+'`/`'w+'`/`'c'`（写时复制） | `memmap.py:215` |
| `_NoValue` 哨兵 | 区分"未传参"与"显式 None"，用于新增关键字参数的向后兼容转发 | `numpy/_globals.py` |
| `_CopyMode` 枚举 | ALWAYS/NEVER/IF_NEEDED，控制 np.copy() 和 np.array() 的拷贝行为 | `numpy/_globals.py` |
| memmap `__array_priority__` | `-100.0`，确保 ufunc 输出不自动升级为 memmap | `memmap.py:213` |

## 5. 错误与重试语义

- **_wrapreduction 失败回退**：当 `getattr(obj, method)` 抛出 `AttributeError` 时，静默回退到 `ufunc.reduce` 路径（`fromnumeric.py:57`），不抛出异常
- **_UFuncNoLoopError**：ufunc 找不到匹配 dtype 的循环时抛出，`__str__` 格式化包含 ufunc 名和输入/输出 dtype（`_exceptions.py:38`）
- **_ArrayMemoryError**：数组分配失败时抛出，`__str__` 计算并友好格式化总字节数（如 "1.5 GiB"）（`_exceptions.py:109`）
- **_display_as_base 装饰器**：所有 ufunc 异常子类的 `__name__` 被改为基类名，用户 traceback 中看到的是基类名（如 TypeError），子类为实现细节可随时移除
- **require 冲突检查**：同时指定 C 和 F 顺序时抛出 `ValueError('Cannot specify both "C" and "F" order')`（`_asarray.py:114`）
- **memmap w+ 模式**：未指定 shape 时抛出 `ValueError("shape must be given if mode == 'w+'")`（`memmap.py:229`）

## 6. 并发细节

- 本叶子为纯 Python 包装层，不直接管理线程或锁。所有实际数值计算在 C 扩展 `_multiarray_umath` 中完成
- **GIL 释放**：C 扩展中的 ufunc 计算释放 GIL 以允许多线程并行；Python 层的 `_methods.py` 桥接函数在调用 ufunc.reduce 时不持有额外锁
- **memmap 线程安全**：memmap 底层使用 OS mmap，多线程读写同一页可能产生竞争；NumPy 不提供额外同步
- **ctypes 桥接**：`_ctypes.data_as()` 存在 CPython bpo-12836 导致的循环引用问题，通过先拷贝指针再附加数组引用的 workaround 规避（`_internal.py:289`）
- **_getintp_ctype 缓存**：`_getintp_ctype.cache` 模块级缓存避免重复 ctypes 类型查找（`_internal.py:248`）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/numeric.py`：Python 级数组创建与操作函数
- `numpy/_core/fromnumeric.py`：规约与排序函数
- `numpy/_core/multiarray.py`：C 扩展再导出薄层
- `numpy/_core/_asarray.py`：require() 函数
- `numpy/_core/_methods.py`：ndarray 方法桥接
- `numpy/_core/_internal.py`：dtype/ctypes/PEP 3118 内部工具
- `numpy/_core/_exceptions.py`：ufunc 异常层级
- `numpy/_core/memmap.py`：memmap 内存映射数组
- `numpy/core/__init__.py`：向后兼容再导出薄层

**Out-of-Scope（不在本仓库源码内）**

- `_multiarray_umath` C 扩展的实际实现（C 源码在 `numpy/_core/src/`，编译为 `.so`）——由 C 扩展域覆盖
- NEP-18 `__array_function__` / NEP-17 `__array_ufunc__` 协议的完整机制——见 `overrides-ufunc-config` 叶子
- 数值类型系统（numerictypes.py、getlimits.py）——见 `numerictypes-getlimits` 叶子
- 数组打印格式化（arrayprint.py）——见 `arrayprint-format` 叶子

## 8. 与相邻子系统交互

- **上游**：用户代码通过 `import numpy as np` 调用本叶子的所有公开函数；`numpy/__init__.py` 从 `_core` 再导出这些符号
- **下游 → C 扩展层**：`multiarray.py` → `_multiarray_umath`（C 扩展）；`_methods.py` → `umath` ufunc reduce；所有数值计算最终落入 C 扩展
- **下游 → overrides**：所有公开函数经 `overrides.array_function_dispatch` 包装，NEP-18 协议检查在 overrides 层完成
- **下游 → numerictypes**：`_internal.py` 的 dtype 操作依赖 `numerictypes` 的类型注册表
- **相邻叶子**：`overrides-ufunc-config` 提供分发装饰器；`numerictypes-getlimits` 提供类型系统；`function-base` 提供更多函数

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **能力缝归组**：本叶子是 NumPy 的纯 Python 包装层，核心能力缝是 **NEP-18 覆盖协议**（`@array_function_dispatch` 装饰器将每个公开函数包装为 `_ArrayFunctionDispatcher`，支持第三方数组库通过 `__array_function__` 接管实现）和 **_wrapreduction 统一规约入口**（纯 ndarray 走 ufunc.reduce，子类走方法调用）
2. **再导出薄层**：`numpy/core/` 是 2.x 向后兼容层，使用 `__getattr__` 惰性转发到 `_core` 并发出弃用警告；`multiarray.py` 从 C 扩展 `_multiarray_umath` re-export 全部符号
3. **哨兵值单例**：`_NoValue` 是模块级单例，通过禁止 reload 保证身份稳定（`_globals.py`），用于区分"未传参"与"显式 None"
4. **性能热路径优化**：`_methods.py` 的 reduce 桥接函数使用位置-only 参数和手动展开 `_NoValue` 检查，避免 kwargs dict 创建开销（`fromnumeric.py:38` 注释引用 gh-31845）

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 数值层架构图 | `numeric-layer-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/numeric-layer-architecture.json`
- 降档原因：无，首次布局调整后即通过 showcase 校验
- 时序图/数据流图：本叶子为纯 Python 包装层，核心调用链已在第 3 节文字化描述，不适用额外时序图（调用链为线性单层转发，无异步/多阶段交互）
