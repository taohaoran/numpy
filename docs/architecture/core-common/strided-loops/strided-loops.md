# 跨步循环与内存重叠基础设施（strided-loops）

> 本文是 `core-common` 域下的叶子子系统文档。域级总览见 `../core-common.md`。
> 本文聚焦跨步内存拷贝/转换内核选择、内存重叠检测、数组赋值辅助，是 ufunc 与广播迭代的底层支撑。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 跨步拷贝函数选择 | `PyArray_GetStridedCopyFn` 按对齐/步长/元素大小选最优拷贝内核 | `multiarray/lowlevel_strided_loops.c.src:381` |
| 通用跨步拷贝 | `_strided_to_strided` 非连续 stride 逐元素拷贝 | `multiarray/lowlevel_strided_loops.c.src:269` |
| 连续拷贝快路径 | `_contig_to_contig` 连续内存走 memcpy | `multiarray/lowlevel_strided_loops.c.src:366` |
| 字节交换拷贝 | `_swap_strided_to_strided` 含字节交换 | `multiarray/lowlevel_strided_loops.c.src:294` |
| 数值转换函数选择 | `PyArray_GetStridedNumericCastFn` 按 dtype 选转换内核 | `multiarray/lowlevel_strided_loops.c.src:1230` |
| N 维到跨步传输 | `PyArray_TransferNDimToStrided` | `multiarray/lowlevel_strided_loops.c.src:1314` |
| 跨步到 N 维传输 | `PyArray_TransferStridedToNDim` | `multiarray/lowlevel_strided_loops.c.src:1447` |
| 掩码传输 | `PyArray_TransferMaskedStridedToNDim` | `multiarray/lowlevel_strided_loops.c.src:1579` |
| 内存重叠检测 | `solve_may_share_memory` 两数组是否共享内存 | `common/mem_overlap.c:758` |
| 内部重叠检测 | `solve_may_have_internal_overlap` 数组自重叠 | `common/mem_overlap.c:849` |
| 丢番图求解 | `solve_diophantine` 用扩展欧几里得判断步长交集 | `common/mem_overlap.c:483` |
| 广播 stride | `broadcast_strides` 按广播规则计算输出 stride | `common/array_assign.c:32` |
| object 类型传输 | any↔object 引用计数拷贝 | `multiarray/dtype_transfer.c:195,308` |
| 零填充/截断拷贝 | zero-pad / truncate 拷贝 | `multiarray/dtype_transfer.c:376,404` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyArrayMethod_StridedLoop` | `array_method.h` | 跨步循环函数指针：按固定 stride 处理 |
| `PyArray_MaskedStridedUnaryOp` | `lowlevel_strided_loops.h:72` | 带掩码的一元跨步操作 |
| `NpyAuxData` | 公共 API | 跨步转换辅助数据（clone/free） |
| `diophantine_term_t` | `mem_overlap.h` | 丢番图方程项（步长项） |
| `solve_diophantine` | `mem_overlap.c:483` | 求解步长是否存在交集 |
| `offset_bounds_from_strides` | `mem_overlap.c:660` | 计算步长数组的偏移边界 |

## 3. 关键调用链

### 3.1 跨步拷贝选择

1. `PyArray_GetStridedCopyFn(aligned, src_stride, dst_stride, itemsize)` 被调用
2. 若 src/dst stride 均连续且对齐 → 选 `_contig_to_contig`（memcpy 快路径）
3. 否则选 `_strided_to_strided`（逐元素步长拷贝）
4. 需字节交换时选 `_swap_strided_to_strided`

### 3.2 内存重叠检测

1. `solve_may_share_memory(a, b)` 被调用（如 ufunc 执行前）
2. `strides_to_terms`（mem_overlap.c:714）将两数组 stride 转为丢番图方程项
3. `diophantine_precompute`（:257）预计算
4. `solve_diophantine`（:483）DFS 搜索是否存在步长交集
5. 返回是否可能重叠；重叠时迭代器用 `COPY_IF_OVERLAP` 复制

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| aligned | 1=指针对齐，0=非对齐 | `GetStridedCopyFn` 参数 |
| src_stride / dst_stride | 固定 stride 或 NPY_MAX_INTP（任意） | 同上 |
| itemsize | 固定元素大小或 0（任意） | 同上 |
| NPY_ITER_COPY_IF_OVERLAP | 迭代器遇重叠自动复制 | NpyIter flags |

## 5. 错误与重试语义

- **重叠检测**：保守返回"可能重叠"，由迭代器决定是否复制
- **转换失败**：strided 循环返回 -1 设置 Python 异常
- **object 引用计数**：object 类型拷贝正确管理 Py_INCREF/Py_DECREF
- 无自动重试

## 6. 并发细节

- **GIL**：strided 拷贝在内层循环中释放 GIL（由调用方 execute_ufunc_loop 控制）
- **无内部锁**：纯内存操作，无 mutex
- **重叠检测**：只读操作，线程安全

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `multiarray/lowlevel_strided_loops.c.src`：跨步拷贝/转换内核选择与实现
- `common/mem_overlap.c`：内存重叠检测
- `common/array_assign.c`：广播 stride 计算
- `multiarray/dtype_transfer.c`：object/zero-pad 等特殊传输

**Out-of-Scope（不在本仓库源码内）**
- NpyIter 广播迭代器（归迭代子系统）
- ufunc 内层计算（见 `../../core-ufunc/ufunc-loops/`）
- SIMD 内建函数（见 `../simd/`）

## 8. 与相邻子系统交互

- **上游 → ufunc-object**：execute_ufunc_loop 经 NpyIter 使用本层拷贝/转换
- **上游 → array 赋值**：数组赋值 `arr[:] = other` 经 broadcast_strides 计算 stride
- **下游 → NpyIter**：提供 `COPY_IF_OVERLAP` 所需的重叠检测

## 9. 语言专项适配口径

1. **.src 模板**：`lowlevel_strided_loops.c.src` 用 repeat 宏按 dtype 实例化。
2. **依赖方向**：`common/` 与 `multiarray/` 互相依赖（mem_overlap 被 multiarray 使用），是 NumPy 内部核心层。
3. **GIL 释放**：纯内存拷贝释放 GIL 并行。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `strided-loops-architecture.html` | architecture | standard |

降档说明：6 组件 6 连线，showcase 标签重叠校验未全过，降 standard 档。JSON IR 源文件位于 `json/` 目录。
