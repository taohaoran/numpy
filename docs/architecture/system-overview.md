# NumPy 系统级架构总览

> 源码基准：git commit `3deeb20adb47f5da91e490ef8feda81114823710`（无 release tag，以 commit SHA 为基准）
> 分析口径：Python + C/Cython 混合项目（按 Python+C++ 混合专项清单执行），纯 Python 子包按纯 Python 专项清单执行
> 文档语言：简体中文（代码标识符、源码路径、命令、flag 名保持原文）

## 一、功能总览

NumPy 是 Python 科学计算的基础包（"the fundamental package for scientific computing with Python"，README 原文），核心能力包括：

| 能力域 | 说明 | 核心源码位置 |
|--------|------|-------------|
| N 维数组对象 | `ndarray` 同类型多维数组，支持步长存储、视图、广播 | `numpy/_core/src/multiarray/` |
| 数据类型系统 | 内置 dtype（int/float/complex/datetime/stringdtype）、用户自定义 dtype、cast 规则 | `numpy/_core/src/multiarray/dtype.c.src`、`dtype_transfer.c` |
| 通用函数 (ufunc) | 逐元素函数，支持广播、reduce/accumulate/outer、类型分派 | `numpy/_core/src/umath/` |
| 线性代数 | 矩阵分解、求解、特征值（基于 vendored lapack_lite） | `numpy/linalg/` |
| 傅里叶变换 | 一维/多维 FFT（基于 vendored pocketfft） | `numpy/fft/` |
| 随机数生成 | 位生成器（MT19937/PCG64/Philox/SFC64）+ 分布采样 + 遗留 RandomState | `numpy/random/` |
| 排序与选择 | quicksort/mergesort/timsort/radixsort + argpartition/argmax | `numpy/_core/src/npysort/` |
| 数组 IO | `.npy`/`.npz` 二进制格式、文本读写、memmap | `numpy/lib/format.py`、`_npyio.py` |
| 掩码数组 | 支持缺失/无效数据的 MaskedArray | `numpy/ma/` |
| 多项式 | 7 种正交多项式族（Polynomial/Chebyshev/Legendre 等） | `numpy/polynomial/` |
| Fortran 接口 | f2py：Fortran 源码 → C 包装 → Python 扩展模块 | `numpy/f2py/` |
| 测试与类型 | pytest 工具、类型存根与 mypy 插件 | `numpy/testing/`、`numpy/_typing/` |

## 二、解决的问题

NumPy 解决了 Python 原生数值计算的核心痛点：

1. **高效多维数组**：Python list 是指针数组，元素为 PyObject，内存占用大、缓存不友好、无法向量化。NumPy `ndarray` 采用连续/步长存储的原始 C 类型缓冲区，配合 SIMD 向量化内核，实现 10–100 倍性能提升。
2. **统一的类型与分派**：不同 dtype（int8/float32/float64/complex128 等）需要不同的计算内核。NumPy 的 ufunc 机制在运行期按 operand dtype 自动选择最优循环内核，并经 SIMD 分派选择 ISA 特化版本（AVX2/AVX512/NEON）。
3. **广播与向量化编程**：无需显式循环即可对不同形状的数组进行逐元素运算，广播规则定义了形状兼容的语义。
4. **C/C++ 生态互操作**：通过 C-API、buffer protocol、f2py 等机制与 C/C++/Fortran 代码无缝集成，是 SciPy/pandas/scikit-learn 等库的基础。
5. **可扩展的覆盖协议**：`__array_function__` / `__array_ufunc__` 允许第三方数组类（duck array）拦截 NumPy API，实现分布式数组、GPU 数组等扩展。

## 三、系统边界

### 上边界（调用方）
- Python 用户脚本、IPython/Jupyter
- 下游科学计算库：SciPy、pandas、scikit-learn、matplotlib、PyTorch（CPU 张量互操作）等
- 调用方式：`import numpy as np`，经顶层 `__init__.py` 聚合再导出全部公开 API

### 下边界（依赖方）
- **CPython 解释器与 C-API**：所有 C 扩展经 CPython C-API 与解释器交互（对象创建、引用计数、GIL）
- **C 标准库 / 数学库**：libm、libc（`numpy/_core/src/npymath/` 提供跨平台数学函数封装）
- **BLAS / LAPACK**（可选，外部）：线性代数后端，运行时动态加载；vendored `lapack_lite` 为无外部 BLAS 时的兜底
- **SIMD 内建指令**（外部，编译器提供）：x86 SSE/AVX2/AVX512、ARM NEON/ASIMD、Power VSX，经 `numpy/_core/src/_simd/` 抽象层分派
- **meson 构建系统**（外部，构建期）：`meson.build` + `meson_cpu/` 定义多产物与 CPU 特性编译

### 内边界（仓库内分层）
```
Python 前端层（numpy/ 顶层 + _core/*.py + lib/ma/polynomial/matrixlib/strings/char/rec）
    ↓ Cython 绑定（_multiarray_umath.pyx、random/*.pyx）
C/Cython 核心层（numpy/_core/src/：multiarray / umath / common / npysort / npymath / _simd）
    ↓ 代码生成（code_generators/ + .src 模板，构建期 meson 触发）
生成的 C 源文件（注册表、循环分发、类型定义、API 导出表）
```
依赖方向：`src/common` → `src/multiarray` / `src/umath` 单向；Python 前端 → Cython 绑定 → C 核心单向，禁止反向依赖。

### 侧边界（vendored 第三方，不拆叶子）
| 组件 | 位置 | 说明 |
|------|------|------|
| pocketfft | `numpy/fft/pocketfft/` | C++ FFT 实现，第三方 vendored |
| lapack_lite | `numpy/linalg/lapack_lite/` | LAPACK 子集，第三方 vendored |
| vendored-meson | `vendored-meson/` | meson 构建系统副本，第三方 vendored |
| tempita | `numpy/_core/code_generators/tempita/` | HTML 模板库（用于 .src 模板），第三方 vendored |
| Highway / x86-simd-sort | `numpy/_core/src/npysort/`、`_simd/` | 部分排序/SIMD 代码参考第三方实现 |

### 不做什么（Out-of-Scope）
- 不提供 GPU/加速器后端（GPU 计算由 CuPy、PyTorch 等外部库实现）
- 不提供分布式数组（由 Dask、CuPy 等实现）
- 不提供自动微分/计算图（由 PyTorch/JAX 实现）
- 不提供稀疏数组（由 SciPy `sparse` 实现）
- 不实现 BLAS/LAPACK 本身（依赖外部或 vendored 子集）
- 不实现 FFT 算法本身（依赖 vendored pocketfft）

## 四、系统架构图

![系统架构图](system-architecture.html)

架构要点：
- **三层分层**：Python API 层 → Cython 绑定层 → C/C++ 核心层
- **ufunc 两层分发**：(1) ufunc 对象层按 dtype/operand 选择 ArrayMethod 与循环函数（`dispatching.cpp`）；(2) 循环内部经 `NPY_CPU_DISPATCH` 按 CPU ISA 选择 SIMD 特化内核（`_simd/` + `npy_cpu_features`）
- **代码生成驱动**：`.src` 模板经 `code_generators/generate_*.py` 在构建期生成 C 源文件（ufunc 注册表、循环分发、类型定义、API 导出表），生成文件禁止手改
- **扩展子包独立 C/C++ 核心**：random（C 位生成器 + 分布采样）、fft（vendored pocketfft C++）、linalg（vendored lapack_lite C++）

## 五、核心时序图（ufunc 调用生命周期）

![ufunc 调用时序图](system-sequence.html)

核心流程：
1. **Python 入口**：`np.add(a, b)` → Python API 层（`umath.py`）→ ufunc 对象 `__call__`
2. **类型解析与分派**：`ufunc_type_resolution` 推断输出 dtype，`dispatching.cpp` 按 operand dtype 选择 ArrayMethod 与循环函数
3. **内核执行**：调用选定 loop 函数 → `NPY_CPU_DISPATCH` 选择 ISA 特化 SIMD 内核 → 迭代所有元素（strided loop）→ 写入输出数组
4. **返回结果**：输出 ndarray 经 Cython 绑定返回 Python 层

## 六、数据流图（数据处理流水线）

![数据处理流水线](system-dataflow.html)

数据流向：
1. **输入**：Python 对象（list/scalar/序列）或 duck array（经 `__array_function__` 覆盖协议）
2. **强制转换**：`array_coercion.c` 发现 shape/dtype，需要 cast 时经 `convert.c` / `dtype_transfer.c` 转换
3. **数组表示**：`ndarray`（`PyArrayObject`，步长存储）+ `dtype` 描述符（`PyArray_Descr`）
4. **分派与计算**：ufunc 分派（`dispatching.cpp`）→ 步长循环（`lowlevel_strided_loops.c`）→ SIMD 内核（AVX2/AVX512/NEON）
5. **输出**：输出 ndarray（新分配或 `out=` 参数）→ Python 结果（ndarray/scalar）

## 七、域与叶子索引

本分析按 12 域 37 叶深入拆解，每个叶子含设计文档级 MD（固定 10 小节）与 archify 架构图（部分含时序图/数据流图）。详见 [README.md](README.md) 三层索引。

| 域 | 叶子数 | 域总览 |
|----|--------|--------|
| core-ndarray（数组对象域） | 6 | [core-ndarray.md](core-ndarray/core-ndarray.md) |
| core-ufunc（通用函数域） | 4 | [core-ufunc.md](core-ufunc/core-ufunc.md) |
| core-common（公共基础设施域） | 3 | [core-common.md](core-common/core-common.md) |
| core-sort（排序域） | 1 | [core-sort.md](core-sort/core-sort.md) |
| core-python-api（Python 前端域） | 6 | [core-python-api.md](core-python-api/core-python-api.md) |
| random（随机数域） | 3 | [random.md](random/random.md) |
| fft（傅里叶变换域） | 1 | [fft.md](fft/fft.md) |
| linalg（线性代数域） | 2 | [linalg.md](linalg/linalg.md) |
| lib（工具库域） | 3 | [lib.md](lib/lib.md) |
| ma-polynomial-matrix（掩码/多项式/矩阵域） | 3 | [ma-polynomial-matrix.md](ma-polynomial-matrix/ma-polynomial-matrix.md) |
| f2py（Fortran 接口域） | 1 | [f2py.md](f2py/f2py.md) |
| support（支撑域） | 4 | [support.md](support/support.md) |

## 八、语言适配口径与质量档位

### 语言适配口径
- **Python + C/Cython 混合**（core-ndarray / core-ufunc / core-common / core-sort / random / fft / linalg）：按 Python+C++ 混合专项清单执行——语言分层与绑定边界（Cython/C-API）、两层分发（ufunc dispatch + SIMD DispatchStub 式特化）、code_generators 代码生成与"生成 vs 手写"边界（.src 模板 → C）、多后端 CPU 分派（meson_cpu）、前端符号面（`__init__.py` 聚合再导出）、依赖方向（`src/common` → `src/multiarray/umath` 单向）
- **纯 Python**（core-python-api / lib / ma / polynomial / matrixlib / f2py / testing / typing）：按纯 Python 专项清单执行——能力缝归组、注册表与工厂、代码生成与 DRY、可选依赖与懒加载、配置驱动分派、`__array_function__`/`__array_ufunc__` 覆盖协议

### 质量档位
- 系统级 3 图均为 **standard** 档（showcase 严格布局校验未全过，降档原因：跨层连接多、组件密集导致边穿节点与标签重叠，经删除低价值边、标签偏移、移动组件后 standard 渲染成功）
- 叶子图：showcase / standard 混合，各叶 MD 第 10 节逐一披露质量档位与降档原因
