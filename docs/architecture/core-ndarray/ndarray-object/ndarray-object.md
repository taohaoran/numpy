# ndarray-object（ndarray-object）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开 ndarray 对象本体的类型定义、创建/析构生命周期、协议实现与内存管理职责，
> 不重复展开 dtype 元数据（见 `../dtype-system/dtype-system.md`）、类型转换（见 `../conversion-coercion/conversion-coercion.md`）、迭代器（见 `../iteration-indexing/iteration-indexing.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`，Python + C 混合，meson 构建。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| ndarray 类型对象 | 定义 `PyArray_Type`（`tp_name="numpy.ndarray"`），绑定 tp_new/tp_dealloc/tp_as_number/tp_as_sequence/tp_as_mapping/tp_as_buffer/tp_richcompare/tp_methods/tp_getset | `numpy/_core/src/multiarray/arrayobject.c:1268` |
| 对象创建 | `array_new`（`ndarray.__new__`）解析 shape/dtype/buffer/offset/strides/order，校验 stride 不越界，委托 `PyArray_NewFromDescr_int` | `arrayobject.c:1115` |
| 核心结构体 | `PyArrayObject_fields`：`data`/`nd`/`dimensions`/`strides`/`base`/`descr`/`flags`/`weakreflist`/`_buffer_info`/`mem_handler` | `numpy/_core/include/numpy/ndarraytypes.h:787` |
| 对象析构 | `array_dealloc` → `_clear_array_attributes`：释放 buffer info、base 引用、data（经 mem_handler 或 free）、dimensions 缓存、descr | `arrayobject.c:466`、`arrayobject.c:386` |
| base 引用管理 | `PyArray_SetBaseObject` 压缩视图链（遇 OWNDATA 或异类型即止），防循环；`PyArray_SetWritebackIfCopyBase` 标记回拷 | `arrayobject.c:155`、`arrayobject.c:109` |
| 回拷解析 | `PyArray_ResolveWritebackIfCopy` 把暂存数组数据回拷到 base；析构时未 Resolve 会 RuntimeWarning | `arrayobject.c:328` |
| 可写性检查 | `PyArray_FailUnlessWriteable` 写前校验；`array_might_be_written` 检测重叠内存写并弃用告警 | `arrayobject.c:589`、`arrayobject.c:554` |
| 富比较 | `array_richcompare` 分发到 ufunc（less/equal/greater 等）；VOID/结构化类型走 `_void_compare` 逐字段 logical_and/or | `arrayobject.c:841`、`arrayobject.c:618` |
| 缓冲区协议 | `array_getbuffer` 实现 `bf_getbuffer`（不实现 bf_releasebuffer，历史兼容） | `numpy/_core/src/multiarray/buffer.c:755`、`buffer.c:900` |
| 方法表 | `array_methods[]`：all/any/argmax/astype/copy/dot/argsort/dumps 等 ~100 个方法，含 `__array_ufunc__`/`__array_function__`/`__array__` 子类钩子 | `numpy/_core/src/multiarray/methods.c:2920` |
| getset 属性 | `array_getsetlist[]`：shape/strides/dtype/flags/data/itemsize/ndim 等属性访问器 | `numpy/_core/src/multiarray/getset.c:749` |
| 内存分配 | `npy_alloc_cache`/`npy_alloc_cache_zero`/`npy_free_cache` 带线程局部缓存；`npy_alloc_cache_dim` 专用维度数组；`PyDataMem_UserNEW/ZEROED/RENEW/FREE` 经 mem_handler | `numpy/_core/src/multiarray/alloc.c:179`、`alloc.h:17` |
| ArrayMethod 抽象层 | `PyArrayMethod_Type`/`PyBoundArrayMethod_Type`：围绕 ufunc loop 的低层 C 函数指针抽象，按 DType 类注册 | `numpy/_core/src/multiarray/array_method.c:568`、`array_method.c:1085` |
| 数组赋值 | `PyArray_CopyObject`：经 `PyArray_DiscoverDTypeAndShape` 发现类型形状后，广播/直接/标量三种路径赋值 | `arrayobject.c:238` |
| array_wrap | `npy_find_array_wrap` 查找子类 `__array_wrap__`/`__array_prepare__` | `numpy/_core/src/multiarray/arraywrap.c` |
| 编译期基础例程 | `compiled_base.c`：monotonic 检查、pack/unpack、类型转换辅助 | `numpy/_core/src/multiarray/compiled_base.c` |
| 抽象 DType 默认描述符 | `abstractdtypes.c`：整数默认描述符（intp）、从 Python long 发现描述符 | `numpy/_core/src/multiarray/abstractdtypes.c` |
| array-api 标准适配 | `array_api_standard.c`：Array API 标准相关钩子 | `numpy/_core/src/multiarray/array_api_standard.c` |
| 生成型类型方法模板 | `arraytypes.c.src`：`@type@`/`@Type@`/`@NAME@` 占位的 C 模板，meson `src_file.process()` 按类型实例化出各数值类型的 scalar 转换/copyswap 等函数 | `numpy/_core/src/multiarray/arraytypes.c.src:196` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyArrayType`（即 `PyArray_Type`） | `arrayobject.c:1268` | ndarray 的 Python 类型对象，C 扩展模块导出 |
| `PyArrayObject_fields` | `ndarraytypes.h:787` | ndarray 内存布局：data 指针、nd、dimensions、strides、base、descr、flags、weakreflist、_buffer_info、mem_handler |
| `PyArray_Chunk` | `ndarraytypes.h:874` | buffer 参数桥接：base/ptr/len/flags |
| `array_as_buffer` | `buffer.c:900` | 缓冲区协议表（仅 getbuffer，无 releasebuffer） |
| `array_methods[]` | `methods.c:2920` | 方法定义表（PyMethodDef 数组） |
| `array_getsetlist[]` | `getset.c:749` | 属性定义表 |
| `array_as_mapping` / `array_as_sequence` | `mapping.c:2206` / `sequence.c:68` | 映射/序列协议表 |
| `PyDataMem_Handler` | `alloc.h:47` | 可插拔内存分配策略句柄（默认 handler） |
| `PyArrayMethod_Type` / `PyBoundArrayMethod_Type` | `array_method.c:568` / `:1085` | ArrayMethod 及其绑定对象类型 |
| `_PyArray_GET_ITEM_DATA` 等稳定 ABI 访问器 | `arrayobject.c:1342` | 稳定 ABI 下按 offset 取结构体字段 |

## 3. 关键调用链

### 3.1 ndarray 创建（`np.ndarray(shape, dtype=...)`）

1. Python 调用 `PyArray_Type.tp_new` = `array_new`（`arrayobject.c:1115`），解析关键字参数 shape/dtype/buffer/offset/strides/order。
2. 若未给 dtype，默认 `NPY_DEFAULT_TYPE`（`arrayobject.c:1153`）。
3. 若给 strides，调用 `PyArray_CheckStrides`（`arrayobject.c:1091`）校验所有轴跨步不越界 buffer。
4. 无 buffer：`PyArray_NewFromDescr_int` 分配并填充字段；若 dtype 含引用（object），`PyArray_SetObjectsToNone` 占位（`arrayobject.c:1201`）。
5. 有 buffer：校验 finalize 回调不存在、buffer 足够大，绑定外部数据指针与 base（`arrayobject.c:1233`）。

### 3.2 ndarray 析构（引用计数归零时）

1. `tp_dealloc` = `array_dealloc`（`arrayobject.c:466`）调用 `_clear_array_attributes(self, NPY_TRUE)`。
2. 释放 `_buffer_info`；若 base 非空且 WRITEBACKIFCOPY，先 `PyArray_ResolveWritebackIfCopy` 回拷（缺失则 RuntimeWarning），再 `Py_CLEAR(base)`（`arrayobject.c:397`）。
3. 若 OWNDATA 且 data 非空：引用型 dtype 先 `PyArray_ClearArray` 清引用；经 mem_handler 调 `PyDataMem_UserFREE`，无 handler 则 `free`（`arrayobject.c:424`）。
4. `npy_free_cache_dim` 释放 dimensions 缓存，`Py_CLEAR(descr)`（`arrayobject.c:459`）。

### 3.3 富比较（`arr1 == arr2`）

1. `tp_richcompare` = `array_richcompare`（`arrayobject.c:841`）按 cmp_op 分支。
2. 普通数值类型走 `PyArray_GenericBinaryFunction` 调对应 ufunc（less/equal/greater 等）。
3. VOID 类型走 `_void_compare`（`arrayobject.c:618`）：有字段时逐字段 subscript 取出，用 `logical_and`（EQ）或 `logical_or`（NE）归并；无字段时按内存字节 `_umath_strings_richcompare`。
4. 若 ufunc 报 `_UFuncNoLoopError` 且为 EQ/NE，降级为广播全 false/true 数组（`arrayobject.c:962`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `NPY_ARRAY_OWNDATA` | 置位表示数组拥有 data，析构时释放 | `ndarraytypes.h` flags |
| `NPY_ARRAY_WRITEABLE` | 可写标志；WRITEBACKIFCOPY 时 base 被清 WRITEABLE | `arrayobject.c:139` |
| `NPY_ARRAY_WRITEBACKIFCOPY` | 暂存数组析构前需 Resolve 回拷 | `arrayobject.c:138` |
| `NPY_ARRAY_WARN_ON_WRITE` | 重叠内存写时弃用告警（每次数组仅一次） | `arrayobject.c:560` |
| `NPY_ARRAY_C_CONTIGUOUS`/`F_CONTIGUOUS` | 连续布局标志 | `ndarraytypes.h` |
| `warn_if_no_mem_policy` | 析构 OWNDATA 数组但无 mem_handler 时告警 | `arrayobject.c:435` |
| `NPY_DEFAULT_TYPE` | 未指定 dtype 时的默认类型 | `arrayobject.c:1154` |
| `NPY_MAXDIMS` | 最大维度数 | 公共头 |

## 5. 错误与重试语义

- 创建路径任一失败（strides 越界、buffer 过小、NewFromDescr_int 失败）均 `goto fail` 清理已分配 descr/dims 并返回 NULL，由 Python 转为异常。
- 析构路径（`array_dealloc`）以 `unraisable=NPY_TRUE` 调用，错误无法抛出时 `PyErr_WriteUnraisable` 打印并视为成功，保证对象一定被释放（`arrayobject.c:367`）。
- WRITEBACKIFCOPY 未 Resolve 即在析构中检测到：发 RuntimeWarning（可转异常），并自增引用防止递归 dealloc，再尝试 Resolve（`arrayobject.c:398`）。
- `PyArray_SetBaseObject` 只允许设置一次；重复设置或循环引用（arr==obj）抛 ValueError（`arrayobject.c:167`、`arrayobject.c:215`）。
- 无重试机制（C 扩展，失败即抛异常）。

## 6. 并发细节

- 本叶子核心为单线程 Python 对象模型，由 GIL 保护对象引用计数；无独立 goroutine/线程。
- `npy_alloc_cache`/`npy_alloc_cache_dim`（`alloc.c:179`/`:215`）使用线程局部缓存减少 malloc 开销，属线程安全分配器。
- 临界区：数组对象字段（data/dimensions/strides）在持有 GIL 时读写；ufunc 执行时可能释放 GIL（见 gil_utils.h），但不触碰本对象字段。
- `mem_handler` 为 per-object 可插拔分配策略句柄（1.22+），支持自定义 malloc/free，无全局锁假设。
- 无 context.Context 传递（C 扩展，用 GIL 错误状态 `PyErr_Occurred()` 传播失败）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- ndarray 类型对象、结构体布局、创建/析构生命周期
- 缓冲区协议、方法表、属性表、富比较
- 内存分配缓存与 mem_handler 接口
- ArrayMethod 抽象层（DType 类方法注册）
- base 引用链压缩与 WRITEBACKIFCOPY 语义

**Out-of-Scope（不在本仓库源码内）**
- 实际数值计算（ufunc loop 内核）——在 umath/ 域
- dtype 元数据定义与 cast 表——见 `dtype-system` 叶子
- 迭代器实现（NpyIter 等）——见 `iteration-indexing` 叶子
- 类型转换/coercion——见 `conversion-coercion` 叶子
- 排序算法——见 `core-sort/sort-algorithms` 叶子
- BLAS/LAPACK、CPU 指令集优化库——外部，不在本仓库源码内

## 8. 与相邻子系统交互

- 上游 → 本叶子：Python 用户层（`np.array`/`np.ndarray`）、`ctors.c`（`PyArray_NewFromDescr` 工厂）、`array_coercion.c`（把任意对象转数组）→ 构造 `PyArrayObject`。
- 本叶子 → 下游：`descr` 指针 → `dtype-system`（`PyArray_Descr`/DTypeMeta）；方法表分发 → umath ufunc（`__array_ufunc__`）；迭代 → `iteration-indexing`（`array_iter`→PySeqIter）；赋值 → `array_assign_array.c`/`array_assign_scalar.c`；内存 → `alloc.c` 的 PyDataMem_Handler（外部可插拔）。
- 双向：base 引用链把视图与数据所有者绑定（`SetBaseObject` 压缩到 OWNDATA 或异类型）。

## 9. 语言专项适配口径（Python + C/Cython 混合）

- **绑定边界**：本叶子是 C 扩展层（`_multiarray_umath` 模块），通过 Python C-API 暴露 `PyArray_Type` 给 Python；`arrayobject.c` 内全部为 C，无 Cython。Python 前端（`numpy/_core/_ufunc_config.py` 等）经 `numpy._multiarray_umath` 调用本层。
- **生成 vs 手写边界**：`arraytypes.c.src`、`arraytypes.h.src` 为手写 C 模板，含 `@type@`/`@Type@`/`@NAME@` 占位符，经 meson `src_file.process()`（`numpy/_core/meson.build:1261`）在构建期按数值类型实例化出 `.c`；产物不入库，禁止手改生成文件。手写文件（arrayobject.c/alloc.c/buffer.c 等）直接编译。
- **两层分发**：本叶子不直接实现算子分发；`array_method.c` 的 `PyArrayMethod` 是围绕 ufunc loop 的函数指针抽象层（按 DType 类注册），真正的算子 × 后端分发在 umath Dispatcher（相邻域）。
- **稳定 ABI 边界**：`_PyArray_GET_ITEM_DATA`（`arrayobject.c:1342`）等按 `offsetof` 取字段，供稳定 ABI 扩展使用；结构体对齐有静态断言（≤8 字节，`arrayobject.c:1314`）。
- **依赖方向**：本叶子依赖 `src/common/`（npy_math、numpyos）与 `dtypemeta`/`convert_datatype`，单向；不反向依赖 Python 前端。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| ndarray-object 架构图 | `ndarray-object-architecture.html` | architecture | standard |
| JSON IR 源 | `json/ndarray-object-architecture.json` | — | — |

降档说明：showcase 因宽节点（PyArrayObject_fields 280px）导致垂直边出侧被自动推断为 left/right、且多处标签与组件重叠；显式标注上下侧后仍有标签 clearance 约束未全过，故降 standard 渲染，已在 JSON 中对垂直边显式 fromSide/toSide、对标签 labelDy 偏移。时序图/数据流图不单独补：本叶子为对象生命周期结构，创建/析构/比较流程已在第 3 节文字化，架构图已表达组件边界。
