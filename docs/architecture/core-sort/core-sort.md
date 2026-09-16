# core-sort（排序域）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`，C++ 算法层。

## 1. 域职责

core-sort 域实现 NumPy 的排序、选择（argpartition）与二分查找（searchsorted）算法内核。它通过 ArrayMethod 机制被上层 `np.sort/argsort/argpartition/searchsorted` 调用，按 dtype 与运行期 CPU 特性分派到标量或 SIMD 内核。

核心代码路径：`numpy/_core/src/npysort/`（C++）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| sort-algorithms | [sort-algorithms.md](sort-algorithms/sort-algorithms.md) | [架构图](sort-algorithms/sort-algorithms-architecture.html) | [数据流](sort-algorithms/sort-algorithms-dataflow.html) | 快排/堆排/timsort/基数排序/选择/二分 + SIMD 分派 |

## 3. 域级机制细节

- **算法矩阵**：introsort（快排→堆排兜底）、timsort（稳定，源自 CPython）、radixsort（整数 O(n)）、quickselect（median-of-medians 兜底）、binsearch。
- **两层分发**：第一层 `NPY_CPU_DISPATCH` 按运行期 CPU 选 SIMD/标量内核；第二层 introsort 内部按递归深度在快排/堆排间切换。
- **多后端 SIMD**：跨架构走 Google Highway（VQSort），x86 专用走 x86-simd-sort；两者 vendored，标注外部。
- **GIL**：大块排序经 gil_utils.h 释放 GIL。

## 4. 域级图

![排序分派数据流](sort-algorithms/sort-algorithms-dataflow.html)
