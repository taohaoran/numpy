# sort-algorithms（sort-algorithms）

> 本文是 `core-sort` 域下的唯一叶子子系统文档。域级总览见 `../core-sort.md`。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| sort/argsort ArrayMethod 注册 | `npysort_methods.cpp`：注册排序 ArrayMethod、resolve_descriptors（ensure_canonical） | `numpy/_core/src/npysort/npysort_methods.cpp` |
| introsort 快排 | `quicksort.hpp`：快排 + 递归过深（2*log n）转 heapsort 的 introsort | `numpy/_core/src/npysort/quicksort.hpp` |
| heapsort | `heapsort.cpp`/`npysort_heapsort.hpp`：比较函数驱动的堆排，兜底最坏情况 | `numpy/_core/src/npysort/heapsort.cpp` |
| timsort | `timsort.hpp`/`timsort_generic.cpp`：稳定排序，源自 CPython listsort | `numpy/_core/src/npysort/timsort.hpp` |
| radixsort | `radixsort.hpp`：整数基数排序，KEY_OF 键推导 | `numpy/_core/src/npysort/radixsort.hpp` |
| selection/argpartition | `selection.hpp`/`selection_methods.cpp`：quickselect，递归过深转 median-of-medians | `numpy/_core/src/npysort/selection.hpp` |
| binsearch | `binsearch.cpp`：searchsorted 二分查找（含 arg 变体） | `numpy/_core/src/npysort/binsearch.cpp` |
| Highway SIMD 快排 | `highway_qsort*.dispatch.cpp`：Google Highway VQSort 跨架构 SIMD 排序 | `numpy/_core/src/npysort/highway_qsort.dispatch.cpp` |
| x86 SIMD 快排 | `x86_simd_qsort*.dispatch.cpp`：x86-simd-sort 专用 qsort/qselect | `numpy/_core/src/npysort/x86_simd_qsort.dispatch.cpp` |
| CPU 分派 | `NPY_CPU_DISPATCH_CURFX`：按运行期 CPU 特性选 SIMD 内核 | 各 `.dispatch.cpp` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `sort_resolve_descriptors` | `npysort_methods.cpp:14` | 排序 ArrayMethod 描述符解析 |
| `NPY_CPU_DISPATCH_CURFX(QSort)` | `highway_qsort.dispatch.cpp` | 按 CPU 分派 QSort 内核 |
| `sort::Quick<reverse>` | `quicksort_generic.hpp` | 通用快排模板 |
| `hwy::VQSortStatic` | highway 外部 | Highway SIMD 排序 |
| `x86simdsortStatic::qsort/qselect` | x86-simd-sort 外部 | x86 SIMD 排序/选择 |
| `binsearch` | `binsearch.cpp` | 二分查找 |

## 3. 关键调用链

### 3.1 np.sort(arr, kind='quick')

1. Python 调 sort ArrayMethod → `npysort_methods.cpp` 的 resolve_descriptors（ensure_canonical）。
2. `NPY_CPU_DISPATCH` 按 dtype + CPU 分派：
   - 整数且 SIMD 可用 → Highway VQSort / x86-simd-sort；
   - 通用 → introsort（quicksort.hpp）：递归深度超 2*log(n) 转 heapsort 兜底 O(NlogN)。
3. kind='stable' → timsort；整数快速路径 → radixsort O(n)。

### 3.2 np.argpartition / searchsorted

1. argpartition → selection.hpp quickselect（median-of-3 pivot，递归过深转 median-of-medians）。
2. searchsorted → binsearch.cpp 二分查找。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `kind` | 'quicksort'/'stable'/'heapsort'/'stable' | np.sort |
| `reverse` | 降序 | 各内核模板 |
| `VQSORT_ENABLED` | Highway SIMD 是否启用 | highway_qsort.dispatch.cpp |
| introsort 阈值 | 2*log(n) | quicksort.hpp |

## 5. 错误与重试语义

- 排序本身纯计算，无 IO；失败多为内存不足（MemoryError）。
- introsort/quickselect 的兜底是算法层面的最坏情况防护（非错误重试）：快排退化转堆排，quickselect 退化转 median-of-medians。
- 无外部重试。

## 6. 并发细节

- 排序为纯函数（原地或索引），无共享可变状态；GIL 在大块排序时释放（gil_utils.h）以允许并行其他操作。
- 无锁；SIMD 内核为单线程数据并行（向量化），非多线程并行。
- 索引排序（argsort）与值排序同算法，多维护一个索引数组。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全部排序/选择/二分算法内核与 CPU 分派
- ArrayMethod 注册

**Out-of-Scope（不在本仓库源码内）**
- Google Highway 库（`hwy/`）——vendored 外部，不拆叶
- x86-simd-sort 库（`x86-simd-sort/`）——vendored 外部，不拆叶
- 上层 np.sort 封装（Python）——属相邻域

## 8. 与相邻子系统交互

- 上游：`np.sort/argsort/argpartition/searchsorted`（Python/umath ArrayMethod 机制）。
- 下游：外部 SIMD 库（Highway/x86-simd-sort）；ndarray-object 提供待排数据。

## 9. 语言专项适配口径（Python + C++ 混合）

- **两层分发**：第一层 `NPY_CPU_DISPATCH` 按运行期 CPU 特性选 SIMD/标量内核；第二层 introsort 内部按递归深度在快排/堆排间切换——两层互补：前者面向硬件特化，后者面向算法最坏情况防护。
- **多后端矩阵**：跨架构走 Highway（VQSort），x86 专用走 x86-simd-sort；两者均 vendored，标注外部。
- **模板特化**：quicksort/timsort/radixsort 为 C++ 模板，按类型实例化；`.dispatch.cpp` 为分派胶水。
- **vendored 边界**：`x86-simd-sort/`、Highway 头文件为第三方，不在本仓库源码内（仅封装调用）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| sort-algorithms 架构图 | `sort-algorithms-architecture.html` | architecture | standard |
| 排序分派数据流 | `sort-algorithms-dataflow.html` | dataflow | standard |
| JSON IR 源 | `json/sort-algorithms-architecture.json`、`json/sort-algorithms-dataflow.json` | — | — |

降档说明：showcase 因 dispatch→simd 垂直边穿 selection 节点，移 simd 至右列后降 standard；数据流图仅一处标签重叠经 labelDy 修复。补充数据流图表达"入口→分派→算法内核→输出"的排序选择管道。
