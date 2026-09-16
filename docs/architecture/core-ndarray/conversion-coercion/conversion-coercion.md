# conversion-coercion（conversion-coercion）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开"把任意 Python 对象转成 ndarray/标量"的强制（coercion）与转换逻辑、标量类型对象体系，
> 不重复展开 ndarray 对象构造（见 `../ndarray-object/ndarray-object.md`）、dtype 描述符（见 `../dtype-system/dtype-system.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 任意对象转数组入口 | `PyArray_FromAny`：把任意 Python 对象强制为 ndarray（支持 min_depth/require_dtype/升维/stride 要求） | `numpy/_core/src/multiarray/ctors.c:1460` |
| 类型形状递归发现 | `PyArray_DiscoverDTypeAndShape`/`_Recursive`：递归展开 list/tuple/嵌套序列，推断 dtype 与形状 | `numpy/_core/src/multiarray/array_coercion.c:1271`、`array_coercion.c:985` |
| 强制缓存 | `coercion_cache_obj`：发现过程中缓存中间转换结果与序列标记，供赋值复用 | `array_coercion.c:662` |
| 内部数组转换器 | `array_converter.c`：处理 `__array_wrap__` 与多参 `result_type` 的内部 helper，替代 asanyarray/asarray | `numpy/_core/src/multiarray/array_converter.c` |
| 对象→数组/标量转换 | `convert.c`：`PyArray_FROM_O` 类转换、标量提取 | `numpy/_core/src/multiarray/convert.c` |
| 转换工具 | `conversion_utils.c`：字符串→type_num、typestr 转换、通用转换辅助 | `numpy/_core/src/multiarray/conversion_utils.c` |
| 标量 C-API | `scalarapi.c`：numpy 标量对象（np.int32 等）的 C-API 访问与构造 | `numpy/_core/src/multiarray/scalarapi.c` |
| 标量类型模板 | `scalartypes.c.src`：`@name@`/`@NAME@`/`@suff@` 占位模板，为每种数值类型实例化标量类型对象与二元运算 | `numpy/_core/src/multiarray/scalartypes.c.src:78` |
| 标量运算回退 | 标量二元运算经 `PyNumber_*` 或 ufunc `n_ops.*` 回退 | `scalartypes.c.src:349` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyArray_FromAny` | `ctors.c:1460` | 任意对象→ndarray 总入口 |
| `PyArray_DiscoverDTypeAndShape` | `array_coercion.c:1271` | 递归发现 dtype 与 shape |
| `coercion_cache_obj` | `array_coercion.c:662` | 发现过程缓存 |
| `PyArray_ArrayConverter`（array_converter） | `array_converter.c` | 多参 wrap/result_type helper |
| `Py@NAME@ArrType_Type` | `scalartypes.c.src:78` | 实例化后的标量类型对象（numpy.int32 等） |
| `gentype_@name@@suff@` | `scalartypes.c.src:326` | 实例化后的标量二元运算函数 |

## 3. 关键调用链

### 3.1 任意对象强制为数组（`np.array([1,2,3])` / `np.asarray(x)`）

1. 入口 `PyArray_FromAny`（`ctors.c:1460`），按 require 标志选择 asarray/astype 路径。
2. 调 `PyArray_DiscoverDTypeAndShape`（`array_coercion.c:1271`）→ `_Recursive`（`array_coercion.c:985`）递归：
   - 若输入已是 ndarray，直接用其 descr/shape；
   - 若为 list/tuple，递归展开元素，累积形状维度；
   - 标量叶子经 `convert.c`/`conversion_utils.c` 转数值，记录到 `coercion_cache_obj`。
3. 发现完成后，按结果 descr+shape 调 `PyArray_NewFromDescr_int` 构造数组（见 ndarray-object 叶子）。

### 3.2 标量对象运算（`np.int32(1) + np.int32(2)`）

1. 标量类型对象由 `scalartypes.c.src` 模板实例化（`Py@NAME@ArrType_Type`，`scalartypes.c.src:78`）。
2. 二元运算 `gentype_@name@@suff@`（`scalartypes.c.src:326`）先 `BINOP_GIVE_UP_IF_NEEDED`，再尝试 `PyNumber_@func@`，最终回退到对应 ufunc（`n_ops.*`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `min_depth`/`max_depth` | FromAny 升降维深度要求 | `ctors.c:1460` |
| `require`（'C'/'F'/'A'/'K'） | 内存连续/对齐要求 | array_converter.c |
| casting 安全级别 | no/equiv/safe/same_kind/unsafe | coercion 链路 |
| `NPY_DEFAULT_TYPE` | 未推断出类型时的默认 | 见 dtype-system |

## 5. 错误与重试语义

- 递归发现时形状不一致（如 `[[1,2],[3]]`）抛 ValueError（inconsistent shape）。
- 类型无法强制（如对象含不可转元素）抛 TypeError，由 FromAny 向上传播。
- 标量运算不匹配时经 `BINOP_GIVE_UP_IF_NEEDED` 返回 NotImplemented，Python 层回退。
- 无重试；C 扩展失败即置错误状态。

## 6. 并发细节

- 单线程 Python 对象模型，GIL 保护；无独立线程。
- coercion_cache_obj 为单次调用的临时对象，函数返回即释放（`npy_free_coercion_cache`）。
- 递归发现无共享可变状态；无锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 任意 Python 对象 → ndarray/标量的强制与递归形状发现
- coercion 缓存、内部 array_converter helper
- 标量类型对象模板与标量运算

**Out-of-Scope（不在本仓库源码内）**
- ndarray 实际构造（NewFromDescr）——见 `ndarray-object` 叶子
- dtype 描述符定义——见 `dtype-system` 叶子
- 实际数值计算（ufunc loop）——umath/ 域
- 第三方 array-like 协议（`__array__` 鸭子类型）——经 FromAny 桥接，外部实现不在本仓库

## 8. 与相邻子系统交互

- 上游：Python 用户层任意对象、`np.array`/`np.asarray`。
- 下游：`PyArray_NewFromDescr_int`（ndarray-object）构造数组；`dtype-system` 查描述符；ufunc（umath）执行标量运算。

## 9. 语言专项适配口径（Python + C 混合）

- **生成 vs 手写**：`scalartypes.c.src`、`arraytypes.c.src` 为手写 C 模板，含 `@name@`/`@NAME@`/`@suff@` 占位，经 meson `src_file.process()`（`numpy/_core/meson.build:1302`/`:1261`）构建期按类型实例化；产物 `.c` 不入库、禁止手改。手写文件（convert.c/array_coercion.c/array_converter.c/conversion_utils.c/scalarapi.c）直接编译。
- **绑定边界**：本叶子是 C 扩展层，经 C-API 暴露 `PyArray_FromAny` 等给扩展作者；Python 侧无对应模块（纯 C）。
- **双轨路径**：已是 ndarray 时走零拷贝快速路径（直接返回），非 ndarray 才走递归发现+构造——避免无谓拷贝。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| conversion-coercion 架构图 | `conversion-coercion-architecture.html` | architecture | standard |
| JSON IR 源 | `json/conversion-coercion-architecture.json` | — | — |

降档说明：showcase 因垂直边（convert→scalarapi）穿过 fromany 节点、array-converter→ndarray-obj 垂直边穿过 dtype-sys 节点，改为对角边并删除被穿边后降 standard。时序图不单独补：递归发现流程已在第 3 节文字化。
