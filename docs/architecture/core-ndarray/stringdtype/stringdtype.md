# stringdtype（stringdtype）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开 NumPy 2.0 新引入的 UTF-8 变长字符串 dtype（StringDType）的实现、存储与转换，
> 不重复展开通用 dtype 元类（见 `../dtype-system/dtype-system.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| StringDType 类实现 | `dtype.c`：新字符串 dtype 的 Python 类、实例创建（na_object/coerce） | `numpy/_core/src/multiarray/stringdtype/dtype.c` |
| 新实例工厂 | `new_stringdtype_instance(na_object, coerce)` | `dtype.c:30` |
| 字符串存储 | `static_string.c/h`：变长字符串存储，小串优化（SSO）+ 可插拔 allocator（malloc/free/realloc） | `numpy/_core/src/multiarray/stringdtype/static_string.h` |
| allocator 生命周期 | `NpyString_new_allocator`/`NpyString_free_allocator`：创建/销毁字符串分配器 | `static_string.h` |
| 大小上限 | `NPY_MAX_STRING_SIZE`：`(1<<8*(sizeof(size_t)-1))-1` | `static_string.h` |
| cast 转换 | `casts.cpp`：StringDType ↔ 其他 dtype 的转换 loop（C++） | `numpy/_core/src/multiarray/stringdtype/casts.cpp` |
| UTF-8 解码 | `utf8_utils.c/h`：`utf8_char_to_ucs4_code`、`num_bytes_for_utf8_character`（branchless） | `numpy/_core/src/multiarray/stringdtype/utf8_utils.c` |
| 缺失值语义 | na_object / coerce：配置缺失对象与强制行为 | `dtype.c:30` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `StringDType`（dtype.c） | `dtype.c` | 新字符串 dtype 的 Python 类 |
| `npy_string_allocator` | `static_string.h` | 字符串分配器句柄 |
| `npy_string_malloc_func/free_func/realloc_func` | `static_string.h` | 可插拔分配函数指针类型 |
| `NpyString_new_allocator` | `static_string.h` | 创建分配器 |
| `utf8_char_to_ucs4_code` | `utf8_utils.h:8` | UTF-8 字节 → UCS4 码点 |
| `num_bytes_for_utf8_character` | `utf8_utils.h:11` | 分支无计算字符字节数 |

## 3. 关键调用链

### 3.1 创建 StringDType 数组

1. `np.array([...], dtype="T")` 经 coercion 发现 dtype，实例化 StringDType（`new_stringdtype_instance`，`dtype.c:30`）。
2. 每个字符串元素由 `static_string` 存储：小串走 SSO，大串走 allocator 堆分配。
3. 与其他 dtype 互转走 `casts.cpp` 注册的 cast loop；解码 UTF-8 用 `utf8_utils.c`。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `na_object` | 缺失值对象 | dtype.c |
| `coerce` | 是否强制转换 | dtype.c |
| allocator | 可插拔 malloc/free/realloc | static_string.h |
| `NPY_MAX_STRING_SIZE` | 单串大小上限 | static_string.h |

## 5. 错误与重试语义

- 无效 UTF-8/分配失败抛 MemoryError/UnicodeDecodeError。
- allocator 缺失时按接口约定未定义行为（头文件明确警告）。
- 无重试。

## 6. 并发细节

- StringDType 对象由 GIL 保护；字符串存储每元素独立堆/SSO。
- allocator 函数指针可自定义，但默认线程安全；无全局锁。
- cast loop 在 ufunc 执行时可能释放 GIL，只读源字符串。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- StringDType 类、static_string 存储与 SSO、allocator、cast、UTF-8 解码

**Out-of-Scope（不在本仓库源码内）**
- 通用 DType 元类/描述符——见 `dtype-system` 叶子
- 旧定长字符串 dtype（'S'/'U'）——见 scalartypes.c.src（conversion-coercion 叶子）
- 第三方 UTF-8 库——本叶子自实现（branchless），不依赖外部

## 8. 与相邻子系统交互

- 上游：Python 用户 `np.dtype("T")`；`dtype-system` 元类注册 StringDType。
- 下游：`casts.cpp` → ufunc cast loop；`utf8_utils.c` 供解码；ndarray-object 持有字符串数组。

## 9. 语言专项适配口径（Python + C/C++ 混合）

- **绑定边界**：dtype.c/static_string.c/utf8_utils.c 为 C，casts.cpp 为 C++（用 raii_utils.hpp 管理资源）；经 DType 元类注册给 Python。
- **新 dtype 体系**：StringDType 是 2.0 新 DType 体系的范例（DTypeMeta 元类 + ArrayMethod 注册），与旧式 'S'/'U' 定长字符串并存。
- **SSO**：static_string 内嵌小串（类似 libstdc++ SSO），避免短串堆分配。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| stringdtype 架构图 | `stringdtype-architecture.html` | architecture | standard |
| JSON IR 源 | `json/stringdtype-architecture.json` | — | — |

降档说明：showcase 因 sd-class→dtype-sys 水平边穿越 static-string，改为 static-string→dtype-sys 后降 standard。时序图不单独补：创建与转换流程已在第 3 节文字化。
