# 线性代数域（linalg）域总览

> 本域包含 NumPy 线性代数系统的 2 个叶子子系统；各叶子详情见对应文档。
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域实现 NumPy 的线性代数功能，基于 BLAS/LAPACK。Python 层提供 solve/inv/eig/svd/cholesky/qr 等高阶 API，C++ 层（umath_linalg.cpp）将 LAPACK 操作包装为 ufunc 风格接口。vendored lapack_lite 是 f2c 转换的 LAPACK/BLAS 实现。按「Python + C++ 混合项目专项分析清单」分析：Python 包装 + vendored C++ 核心的边界、全局锁串行化、后端切换。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|------------|
| linalg-core | [linalg-core.md](linalg-core/linalg-core.md) | [架构图](linalg-core/linalg-core-architecture.html) | C++ ufunc 包装层 + vendored lapack_lite |
| linalg-python | [linalg-python.md](linalg-python/linalg-python.md) | [架构图](linalg-python/linalg-python-architecture.html) | Python 高阶线性代数 API |

## 3. 域级机制细节

- **后端切换**：编译期宏 `HAVE_EXTERNAL_LAPACK` 切换外部 BLAS/LAPACK 或 vendored lapack_lite
- **全局锁**：无外部 BLAS 时使用 PyMutex/PyThread_type_lock 串行化 lapack_lite 调用（非线程安全）
- **类型提升**：Python 层 `_commonType` 统一 float32→float64 提升规则
- **vendored 边界**：lapack_lite 是 f2c 从 Fortran 77 转换的 LAPACK/BLAS，标注"第三方 vendored，不在本项目原创源码内"

## 4. 域级图

本域各叶子独立出图，无域级架构图。
