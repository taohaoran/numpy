# ufunc 对象与归约子系统（ufunc-object）

> 本文是 `core-ufunc` 域下的叶子子系统文档。域级总览见 `../core-ufunc.md`。
> 本文聚焦 ufunc 对象的 Python 调用入口、类型提升分发的对象层编排、以及 reduce/accumulate/reduceat/outer 归约族的实现，不重复展开算子内核循环（见 `../ufunc-loops/`）与分发匹配算法（见 `../ufunc-dispatch/`）。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| ufunc 对象类型 | `PyUFuncObject` 结构体：持有 nin/nout、functions/data 函数指针表、ntypes、types 类型表、identity 归约单位、`_loops` 注册表与 `_dispatch_cache` | `numpy/_core/include/numpy/ufuncobject.h:103-239` |
| Python 调用入口 | `ufunc_generic_fastcall` 解析 out/where/order/axis 等参数，处理标量快路径与参数个数校验 | `umath/ufunc_object.c:4882` |
| 向量调用 | `ufunc_generic_vectorcall` 作为 vectorcall 槽函数转发 | `umath/ufunc_object.c:5243` |
| 标量快路径 | `try_trivial_scalar_call` 单输入标量时绕过迭代器直接调用 | `umath/ufunc_object.c:4747` |
| 平凡单输出快路径 | `try_trivial_single_output_loop`  contiguous 单输出时跳过缓冲 | `umath/ufunc_object.c:879` |
| 普通循环执行 | `execute_ufunc_loop` 建 NpyIter、获取 strided_loop、逐块执行、检查浮点错误 | `umath/ufunc_object.c:1089` |
| 类型解析 | `resolve_descriptors` 将输入 dtype 提升为签名 dtype | `umath/ufunc_object.c:96,4476` |
| 归约 reduce | `PyUFunc_Reduce` 选 ArrayMethod、尝试 contiguous 快路径、否则走 ReduceWrapper | `umath/ufunc_object.c:2884` |
| 累积 accumulate | `PyUFunc_Accumulate` 沿单轴累积，需三描述符等价 | `umath/ufunc_object.c:2946` |
| 归约索引 reduceat | `PyUFunc_Reduceat` 按索引数组分段归约 | `umath/ufunc_object.c:3359` |
| 外积 outer | `ufunc_outer` 两数组外积展平为二进 ufunc 调用 | `umath/ufunc_object.c:5970` |
| 归约包装器 | `PyUFunc_ReduceWrapper` 建归约迭代器、处理 identity 初始化、调用 reduce_loop | `umath/reduction.c:179` |
| 归约内层循环 | `reduce_loop` 将 (N+1) 迭代器操作数展开为 (N+1)->N strided_loop 参数 | `umath/ufunc_object.c:2614` |
| contiguous 归约快路径 | `try_reduce_contiguous` 全归约连续对齐输入时直调 strided_loop | `umath/ufunc_object.c:2724` |
| 通用 gufunc 调用 | `PyUFunc_GeneralizedFunctionInternal` 处理 core 签名维度 | `umath/ufunc_object.c:1751` |
| 创建 ufunc | `PyUFunc_FromFuncAndDataAndSignatureAndIdentity` 注册函数指针表 | `umath/ufunc_object.c:5359` |
| 模块命名空间 | `umath.py` 从 `_multiarray_umath` 再导出全部 ufunc 名称 | `numpy/_core/umath.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyUFuncObject` | `ufuncobject.h:103` | ufunc Python 对象；持有函数指针数组 `functions`、`data`、类型表 `types`、ntypes、`_loops`（OrderedDict）、`_dispatch_cache` |
| `PyUFuncGenericFunction` | `ufuncobject.h` 公共 API | 旧式 1D 循环函数指针签名 `(char** args, npy_intp* dims, npy_intp* steps, void* func)` |
| `PyArrayMethod_Context` | `array_method.h` | 新版调用上下文：caller(ufunc)、method(ArrayMethod)、descriptors |
| `PyArrayMethod_StridedLoop` | `array_method.h` | 新版 strided 循环函数指针：按固定 stride 逐块处理 |
| `PyUFunc_TypeResolutionFunc` | `ufuncobject.h:176` | 旧式类型解析回调 |
| `NpyIter` | `nditer` | 广播迭代器：处理 shape 对齐、缓冲、内部/外部循环 |
| `npy_extobj` | `extobj.h` | 错误对象：bufsize、errmask 位掩码、pyfunc 回调 |
| identity 常量 | `ufuncobject.h:277-303` | `PyUFunc_Zero/One/MinusOne/None/ReorderableNone/IdentityValue` 决定归约可重排性 |

## 3. 关键调用链

### 3.1 普通 ufunc 调用（二进加法为例）

1. Python 调用 `np.add(a, b)` → 进入 `ufunc_generic_vectorcall`（`ufunc_object.c:5243`）
2. `ufunc_generic_fastcall`（:4882）解析位置/关键字参数，提取 out/where/order
3. 构造操作数组与 operand DType 数组
4. 调用 `promote_and_get_ufuncimpl`（dispatching.cpp:1138）做类型提升与分发——遍历 `ufunc->_loops` 注册表，匹配最佳 ArrayMethod（详见 `../ufunc-dispatch/`）
5. 调用 `execute_ufunc_loop`（:1089）：
   - `validate_casting` 校验转换规则
   - `NpyIter_AdvancedNew` 建广播迭代器（`NPY_ITER_BUFFERED | GROWINNER` 等）
   - `context->method->get_strided_loop` 获取固定 stride 的 strided_loop
   - 释放 GIL（`NPY_BEGIN_THREADS_THRESHOLDED`），`do { strided_loop(...) } while iternext`
   - 结束后 `_check_ufunc_fperr` 检查硬件浮点状态
6. 返回结果数组

### 3.2 reduce 归约调用

1. Python 调用 `np.add.reduce(a, axis=0)` → `PyUFunc_Reduce`（:2884）
2. `_get_bufsize_errmask` 从 TLS 错误对象取 bufsize/errormask
3. `reducelike_promote_and_resolve`（:2476）做归约专用类型提升
4. `try_reduce_contiguous`（:2724）尝试全归约连续对齐快路径
5. 慢路径：`PyUFunc_ReduceWrapper`（reduction.c:179）：
   - 校验可重排性（非 reorderable 只允许单轴）
   - 建归约迭代器（`NPY_ITER_REDUCTION_AXIS` 标记归约轴）
   - 无 identity 时 `PyArray_CopyInitialReduceValues` 复制首元素
   - 调用 `reduce_loop`（:2614）：将 (N+1) 迭代器操作数展开为 (N+1)->N strided_loop 参数，逐块归约

### 3.3 accumulate 累积调用

1. `PyUFunc_Accumulate`（:2946）类型提升后建迭代器
2. 逐元素沿轴累积，输出与输入同 shape
3. 需三个描述符（输入/输出/中间）类型等价

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `bufsize` | 迭代器缓冲区大小，默认 `NPY_BUFSIZE`，经 `np.errstate(bufsize=)` 调整，需 5..10e6 且 16 倍数 | `extobj.c:231-249` |
| `errmask` | 四类浮点错误（divide/over/under/invalid）各 3bit，默认 underflow=ignore 其余=warn | `extobj.c:41-44` |
| `where` | 布尔掩码选择性执行 | `ufunc_object.c:598` |
| `order` | 数组内存布局顺序 C/F/A/K | `ufunc_object.c` 参数解析 |
| `casting` | 转换规则 safe/same_kind/unsafe | `validate_casting:1054` |
| `dtype`/`signature` | 固定输出 dtype 签名，覆盖提升结果 | `dispatching.cpp` 步骤1 |
| `UFUNC_NO_FLOATINGPOINT_ERRORS` | ufunc 不产生浮点错误时跳过 fperr 检查以提速 | `ufuncobject.h:261` |

## 5. 错误与重试语义

- **浮点错误**：循环执行后 `_check_ufunc_fperr`（`extobj.c:538`）读取硬件 FPU 状态寄存器，按 errmask 分发到 ignore/warn/raise/call/log 六种处理（`_error_handler` :390）。raise 时设置 `PyExc_FloatingPointError`。
- **类型不匹配**：`resolve_implementation_info` 无匹配 loop 时抛 `TypeError`；promoter 递归提升失败时回退。
- **归约空轴**：无 identity 的归约遇空轴抛 ValueError；有 identity 时初始化为单位值。
- **不可重排多轴归约**：非 reorderable ufunc 多轴归约抛 ValueError（`reduction.c:212-218`）。
- **迭代器分配失败**：`NpyIter_AdvancedNew` 返回 NULL 即失败，清理已分配描述符。
- 无自动重试机制；错误通过 Python 异常上抛。

## 6. 并发细节

- **GIL 释放**：`execute_ufunc_loop` 中若 method 不要求 Python API（`!(flags & NPY_METH_REQUIRES_PYAPI)`）且数组足够大（`NPY_BEGIN_THREADS_THRESHOLDED`），释放 GIL 执行内层循环。
- **归约线程**：`reduce_loop` 同样在 `needs_api == 0` 时释放 GIL。
- **线程安全**：extobj 经 Python `ContextVar`（`npy_extobj_contextvar`）实现线程/协程隔离；`fetch_curr_extobj_state`（`extobj.c:118`）每次调用取当前上下文副本。
- **无内部锁**：ufunc 执行本身无 mutex；NpyIter 内部状态为单次调用局部。
- **TLS 全局**：bufsize/errmask 经 `_get_bufsize_errmask` 从 extobj 提取。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `umath/ufunc_object.c`：ufunc 对象方法、调用入口、归约四操作
- `umath/ufunc_type_resolution.c`：旧式类型解析
- `umath/legacy_array_method.c`：旧式 loop 适配为 ArrayMethod
- `umath/reduction.c`：归约通用包装器
- `umath/legacy_array_method.h`、`reduction.h`
- `numpy/_core/umath.py`：Python 命名空间再导出

**Out-of-Scope（不在本仓库源码内）**
- 实际算子内核计算（见 `../ufunc-loops/`）
- 类型提升匹配算法细节（见 `../ufunc-dispatch/`）
- NpyIter 迭代器内部实现（归 multiarray 迭代子系统）
- 浮点错误处理对象机制（见 `../ufunc-error/`）
- SIMD CPU 分派（见 `../../core-common/simd/`）

## 8. 与相邻子系统交互

- **上游**：Python 层 `numpy/_core/_ufunc_config.py`（seterr/errstate）、`numpy/_core/umath.py`；用户代码经 `np.add(...)` 等调用
- **下游 → ufunc-dispatch**：本叶子调用 `promote_and_get_ufuncimpl` 做 dtype 提升与 ArrayMethod 匹配
- **下游 → ufunc-loops**：选中 ArrayMethod 后经 `get_strided_loop` 获取实际内核函数指针
- **下游 → ufunc-error**：循环结束后调用 `_check_ufunc_fperr` 检查浮点异常
- **下游 → strided-loops**：NpyIter 与 strided copy/transfer 支撑广播与缓冲

## 9. 语言专项适配口径

本项目为 Python + C 混合项目（Python+Cython 核心），按混合专项清单执行：

1. **绑定边界**：C 实现的 `PyUFunc_Type` 类型通过 `PyModule_AddObject` 暴露为 Python ufunc 对象；`umath.py` 纯做命名空间再导出，无逻辑。
2. **两层分发**：本叶子负责第一层（ufunc 对象层按 dtype 选 ArrayMethod）；第二层（loop 内部 SIMD ISA 特化）见 `../ufunc-loops/` 与 `../../core-common/simd/`。
3. **代码生成**：`funcs.inc.src`、`loops.c.src` 等 `.src` 模板经 C 预处理器 repeat 宏（`/**begin repeat**/`）展开为多类型实例化 C 文件，meson 构建时生成。
4. **依赖方向**：`umath/` 依赖 `common/`（lowlevel_strided_loops、mem_overlap）与 `multiarray/`（nditer、dtype_transfer），单向不反向。
5. **GIL 释放**：内层数值循环释放 GIL 以支持多线程，object 类型 loop 保留 GIL（`NPY_METH_REQUIRES_PYAPI`）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `ufunc-object-architecture.html` | architecture | standard |
| 调用时序图 | `ufunc-object-sequence.html` | sequence | standard |

降档说明：架构图跨两层边界、含 9 组件 9 连线，showcase 严格布局校验（端点方向/标签重叠/边界走线）多次迭代未全过，降 standard 档渲染。时序图因参与者较多、showcase 要求消息跨度≥60px，降 standard 档。JSON IR 源文件位于 `json/` 目录。
