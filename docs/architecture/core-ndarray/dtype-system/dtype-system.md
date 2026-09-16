# dtype-system（dtype-system）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开 dtype（数据类型描述符）对象本身、DType 元类、cast 安全性表与类型提升逻辑，
> 不重复展开 ndarray 对象持有 descr 的方式（见 `../ndarray-object/ndarray-object.md`）、
> 标量转换（见 `../conversion-coercion/conversion-coercion.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| dtype 对象类型 | `PyArrayDescr_Type`：ndarray 元素类型描述符的 Python 类型，提供 dtype() 构造、属性（kind/type/byteorder/itemsize/str/name/fields） | `numpy/_core/src/multiarray/descriptor.c:3961` |
| 描述符结构体 | `PyArray_Descr_fields`：typeobj/kind/type/byteorder/_former_flags/type_num/flags/elsize/alignment/metadata/hash | `numpy/_core/include/numpy/ndarraytypes.h:619` |
| DType 元类 | `PyArray_DTypeMeta`：2.0 新 dtype 体系的元类，管理 parametric/nonparametric dtype、默认描述符工厂、type 发现 | `numpy/_core/src/multiarray/dtypemeta.c:185`、`dtypemeta.c:521` |
| 默认描述符工厂 | 各 dtype 的 `default_descr`：整数→intp、字符串/unicode→可变、datetime/timedelta、void 的默认描述符 | `dtypemeta.c:679`/`:705`/`:712`/`:728` |
| cast 安全常量表 | `_npy_can_cast_safely_table[21][21]`：编译期常量，FITS/UFITS/IFITS 宏按位宽判定安全转换 | `numpy/_core/src/multiarray/can_cast_table.h` |
| cast 循环注册 | `convert_datatype.c`：内置类型对的 cast 循环（cast 实现）注册与分发 | `numpy/_core/src/multiarray/convert_datatype.c` |
| dtype 传输函数 | `PyArray_GetDTypeTransferFunction`：按 from/to descr 与 cast 安全性选择低阶 stride 传输 loop | `numpy/_core/src/multiarray/dtype_transfer.c` |
| 类型提升 | `common_dtype.c`：`PyTypeNum_is_equal`、common dtype 计算（result_type 语义） | `numpy/_core/src/multiarray/common_dtype.c:49`/`:205` |
| 旧 dtype 实现 | `legacy_dtype_implementation.c`：1.x 旧式 dtype 行为兼容层 | `numpy/_core/src/multiarray/legacy_dtype_implementation.c` |
| 公共 dtype API | `public_dtype_api.c`：对外开放的 dtype 注册/查询 C-API | `numpy/_core/src/multiarray/public_dtype_api.c` |
| dtype 哈希 | `hashdescr.c`：描述符哈希计算 | `numpy/_core/src/multiarray/hashdescr.c` |
| Python 侧名字/字符串 | `_dtype.py`：`_kind_to_stem` 映射、`__str__`/`__repr__`（字段/子数组/普通）字符串化 | `numpy/_core/_dtype.py:8` |
| dtype traversal | `dtype_traversal.c`：GC traverse/clear 描述符（含 metadata） | `numpy/_core/src/multiarray/dtype_traversal.c` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyArray_Descr_fields` | `ndarraytypes.h:619` | dtype 实例内存布局 |
| `PyArray_DTypeMeta` | `ndarraytypes.h` / `dtypemeta.c` | dtype 的元类（dtype 类本身是元类实例） |
| `PyArrayDescr_Type` | `descriptor.c:3961` | dtype 对象的类型对象 |
| `_npy_can_cast_safely_table` | `can_cast_table.h` | 21×21 编译期 cast 安全表 |
| `PyArray_GetDTypeTransferFunction` | `dtype_transfer.c` | 按 cast 安全性选 stride 传输 loop |
| `PyDTypeMeta_DTypeFromDType` / common dtype 计算 | `common_dtype.c:49` | 类型提升公共 dtype |
| `PyArray_DTypeMeta_default_descr` | `dtypemeta.c:679` | 非参数化 dtype 的默认描述符 |

## 3. 关键调用链

### 3.1 dtype 创建（`np.dtype('int32')` / `np.dtype(np.int32)`）

1. Python 调 `PyArrayDescr_Type.tp_new`，经 descriptor.c 解析输入（字符串/类型/已有 descr）。
2. 若为类型字符串，`_dtype.py` 的 `__str__`/名字映射参与人类可读字符串；C 侧按 type_num 查 `PyArray_DescrFromType`。
3. 实例化 `PyArray_Descr_fields`，填 typeobj/kind/type/byteorder/type_num/elsize/alignment/metadata。
4. DType 元类（`PyArray_DTypeMeta`）为该 dtype 类管理默认描述符工厂（`dtypemeta.c:164` legacy_dtype_default_new）。

### 3.2 cast 安全性判定（`np.can_cast(a, b)`）

1. 查 `_npy_can_cast_safely_table[type_from][type_to]`（`can_cast_table.h`）快速路径。
2. 表项由 FITS/UFITS/IFITS 宏按类型位宽编译期展开；参数化 dtype 走 DTypeMeta 运行期逻辑。
3. 实际数据传输由 `PyArray_GetDTypeTransferFunction`（`dtype_transfer.c`）按 cast 安全级别选择 loop，cast 实现注册在 `convert_datatype.c`。

### 3.3 类型提升（`np.result_type(a, b)`）

1. `common_dtype.c` 计算两 descr 的公共 dtype（`common_dtype.c:49`/`:205`）。
2. 结果 descr 供 ufunc 循环选择与数组构造使用。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `PyArray_Descr.flags` | parametric/object/has_metadata 等实例标志位 | `ndarraytypes.h:642` |
| `NPY_NTYPES_LEGACY` | 内置类型总数（21） | `can_cast_table.h` |
| cast 安全级别 | no/no safe/equiv/safe/same_kind/unsafe | dtype_transfer.c |
| dtype metadata | 可选元数据 dict | `PyArray_Descr_fields.metadata` |

## 5. 错误与重试语义

- 未知类型字符串/字段名在 dtype 构造时抛 TypeError/ValueError；`_dtype.py` 未知 kind 抛 RuntimeError（`_dtype.py:38`）。
- cast 不允许（can_cast 失败）时 ufunc 报 `UFuncNoLoopError`，上层 `array_richcompare` 等捕获降级（见 ndarray-object 叶子）。
- 无重试；C 扩展失败即置 Python 错误状态。

## 6. 并发细节

- dtype 对象为 Python 对象，由 GIL 保护；can_cast_table 为编译期只读常量，无线程安全问题。
- DType 元类的默认描述符工厂在 import 期初始化，运行期只读。
- 无独立线程；cast loop 在 ufunc 执行时可能释放 GIL，但只读 descr。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- dtype/Descr 对象与 DTypeMeta 元类
- cast 安全常量表、cast 循环注册、传输函数选择
- 类型提升（common dtype）、Python 侧字符串化

**Out-of-Scope（不在本仓库源码内）**
- 实际 cast 数据循环的 SIMD/后端内核——umath/ 域
- 标量对象（np.int32 等）实现——见 `conversion-coercion` 叶子
- datetime/timedelta 专属语义——见 `datetime` 叶子
- stringdtype (UTF-8)——见 `stringdtype` 叶子

## 8. 与相邻子系统交互

- 上游：Python 用户层 `np.dtype(...)`、`_dtype.py`（字符串）、ndarray-object 持有 descr 指针。
- 下游：`convert_datatype.c` → umath ufunc cast loop；`dtype_transfer.c` → 低阶 stride loop（`lowlevel_strided_loops`）；`common_dtype.c` 供 ufunc 类型提升。

## 9. 语言专项适配口径（Python + C 混合）

- **绑定边界**：描述符对象在 C 侧定义（descriptor.c/dtypemeta.c），Python 侧仅 `_dtype.py` 处理字符串化；C-API 经 `numpy/_core/include/npy_*.h` 暴露。
- **两层分发**：cast 安全表是编译期静态路由（`can_cast_table.h`），实际传输 loop 选择是运行期查表（`dtype_transfer.c`）——两层互补：前者给编译期 generic 代码用，后者给运行期 cast 用。
- **生成 vs 手写**：`arraytypes.c.src` 等模板经 meson 实例化（见 ndarray-object 叶子第 9 节）；dtype 核心文件（descriptor.c/dtypemeta.c）为手写 C。
- **依赖方向**：`src/common` → descriptor/dtypemeta → convert_datatype，单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| dtype-system 架构图 | `dtype-system-architecture.html` | architecture | standard |
| JSON IR 源 | `json/dtype-system-architecture.json` | — | — |

降档说明：showcase 因相邻组件水平边间隙不足（descr-struct 与 ndarray-obj 间仅 50px）导致标签越界，去掉该边文字标签后降 standard。时序图/数据流图不单独补：cast 判定流程已在第 3 节文字化，架构图已表达描述符与 cast 子系统关系。
