# 通用函数域（core-ufunc）域总览

> 本域包含 NumPy 通用函数（ufunc）的对象层、内核循环、类型分发与错误处理四个叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

core-ufunc 域实现 NumPy 的通用函数（ufunc）体系：将 Python 级数组操作编译为逐元素/逐块的 C 内核调用。域内分两层分发——

- **第一层（对象层）**：按操作数 dtype 提升并匹配到最佳 `PyArrayMethod` 实现（`ufunc-dispatch`）
- **第二层（内核算子）**：选中的内核经 SIMD 按 CPU ISA 选择优化版本（`ufunc-loops`）

ufunc 对象本身管理 Python 调用入口、归约四操作（reduce/accumulate/reduceat/outer）与执行编排（`ufunc-object`）；浮点错误处理经 ContextVar 隔离（`ufunc-error`）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流 | 职责一句话 |
|------|------|--------|-------------|------------|
| ufunc-object | [ufunc-object.md](ufunc-object/ufunc-object.md) | [架构图](ufunc-object/ufunc-object-architecture.html) | [时序图](ufunc-object/ufunc-object-sequence.html) | ufunc 对象、调用入口、归约四操作 |
| ufunc-loops | [ufunc-loops.md](ufunc-loops/ufunc-loops.md) | [架构图](ufunc-loops/ufunc-loops-architecture.html) | — | 内核循环模板、SIMD 特化、.src 代码生成 |
| ufunc-dispatch | [ufunc-dispatch.md](ufunc-dispatch/ufunc-dispatch.md) | [架构图](ufunc-dispatch/ufunc-dispatch-architecture.html) | — | dtype 提升与 ArrayMethod 匹配算法 |
| ufunc-error | [ufunc-error.md](ufunc-error/ufunc-error.md) | [架构图](ufunc-error/ufunc-error-architecture.html) | — | 浮点错误状态配置与运行时处理 |

## 3. 域级机制细节

### 两层分发模型

1. **对象层分发（ufunc-dispatch）**：`promote_and_get_ufuncimpl` 遍历 `ufunc->_loops` 注册表，按 dtype 子类匹配最佳 ArrayMethod，promoter 可递归提升。结果缓存于 `_dispatch_cache`。
2. **内层 ISA 特化（ufunc-loops + simd）**：ArrayMethod 的 strided_loop 经 `NPY_CPU_HAVE` 运行时选择 AVX2/AVX512/SSE/NEON 编译版本，由 meson_cpu 多版本编译保证。

### 归约统一抽象

reduce/accumulate/reduceat 共用 `PyUFunc_ReduceWrapper`（reduction.c），通过 `identity` 字段决定多轴可重排性；`reduce_loop` 将 (N+1) 迭代器操作数展开为 (N+1)->N strided_loop 参数。

### 错误状态线程隔离

extobj 经 Python `ContextVar` 存储 bufsize/errmask/pyfunc，线程/协程独立；ufunc 循环结束后统一检查硬件 FPU 状态并按配置处理。
