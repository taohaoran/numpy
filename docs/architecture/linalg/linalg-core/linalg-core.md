# 线性代数核心（linalg-core）

> 本文是 `linalg` 域下的叶子子系统文档。域级总览见 `../linalg.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| LAPACK ufunc 包装 | umath_linalg.cpp 将 LAPACK 线性代数操作包装为 ufunc 风格接口（solve、inv、eig、svd、cholesky 等） | `numpy/linalg/umath_linalg.cpp` |
| vendored LAPACK | lapack_lite/：f2c 从 Fortran 转换的 LAPACK/BLAS 实现，支持 s/d/c/z 四种精度 | `numpy/linalg/lapack_lite/` |
| Python C 扩展模块 | lapack_litemodule.c：将 lapack_lite 导出为 Python C 扩展 | `numpy/linalg/lapack_litemodule.c` |
| 全局锁 | 无外部 BLAS 时使用 PyMutex/PyThread_type_lock 串行化 lapack_lite 调用 | `umath_linalg.cpp:43-49` |
| 后端切换 | HAVE_EXTERNAL_LAPACK 编译宏切换外部 BLAS/LAPACK 或 vendored lapack_lite | `umath_linalg.cpp:43` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `lapack_lite_lock` | `umath_linalg.cpp:45-48` | 全局互斥锁，串行化非线程安全的 lapack_lite 调用 |
| `f2c_blas.c`/`f2c_d_lapack.c` 等 | `lapack_lite/` | f2c 转换的 BLAS/LAPACK 实现，按精度分文件（s=single, d=double, c=complex, z=complex16） |
| `python_xerbla.c` | `lapack_lite/` | LAPACK 错误处理回调（xerbla），将 Fortran 错误转为 Python 异常 |

## 3. 关键调用链

### 3.1 线性求解调用链

1. Python `_linalg.solve(a, b)` 调用 umath_linalg 的 solve ufunc
2. umath_linalg.cpp 检查 HAVE_EXTERNAL_LAPACK：有则调用外部 BLAS，无则获取 lapack_lite_lock
3. 调用对应精度的 LAPACK 函数（如 dgesv 求解 double 矩阵方程）
4. 结果写回输出数组，释放锁

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `HAVE_EXTERNAL_LAPACK` | 编译期宏；定义时使用外部 BLAS/LAPACK，未定义时使用 vendored lapack_lite | `umath_linalg.cpp:43` |
| `UMATH_LINALG_USE_PYMUTEX` | Python 3.13+ 且非 Limited API 时使用 PyMutex，否则用 PyThread_type_lock | `umath_linalg.cpp:36-40` |
| `NPY_TARGET_VERSION` | NPY_2_1_API_VERSION | `umath_linalg.cpp:11` |

## 5. 错误与重试语义

- **LAPACK 错误**：python_xerbla.c 捕获 Fortran LAPACK 错误，转换为 Python 异常（LinAlgError）
- **奇异矩阵**：solve/inv 遇到奇异矩阵时抛出 LinAlgError
- **无重试机制**：LAPACK 调用失败直接抛出异常

## 6. 并发细节

- **全局锁串行化**：无外部 BLAS 时，所有 lapack_lite 调用通过全局锁串行化（lapack_lite 非线程安全）
- **GIL 释放**：LAPACK 计算期间释放 GIL，允许多线程在不同外部 BLAS 调用上并行
- **外部 BLAS**：有外部 BLAS 时不使用全局锁（假设外部 BLAS 线程安全）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/linalg/umath_linalg.cpp`：C++ ufunc 包装
- `numpy/linalg/lapack_litemodule.c`：Python C 扩展模块
- `numpy/linalg/lapack_lite/`：vendored LAPACK/BLAS（f2c 转换）

**Out-of-Scope（不在本项目原创源码内）**

- `lapack_lite/` 中的 f2c 转换代码是 vendored 第三方（原始 LAPACK/BLAS 为 Fortran 77），在边界表标注"第三方 vendored，不在本项目原创源码内"
- 外部 BLAS/LAPACK 库（OpenBLAS/MKL/Accelerate 等）——不在本仓库源码内

## 8. 与相邻子系统交互

- **上游 → linalg-python**：`_linalg.py` Python 层调用 umath_linalg 的 ufunc
- **下游 → LAPACK**：umath_linalg → lapack_lite（vendored）或外部 BLAS/LAPACK

## 9. 语言专项适配口径

本叶子按「Python + C++ 混合项目专项分析清单」分析：

1. **vendored 边界**：lapack_lite 是 f2c 从 Fortran 77 自动转换的 LAPACK/BLAS 实现，属于第三方 vendored 代码，不深入分析其内部数值算法
2. **全局锁**：lapack_lite 非线程安全，umath_linalg 使用全局锁串行化——这是 vendored C 库线程安全限制的典型应对
3. **后端切换**：编译期宏 HAVE_EXTERNAL_LAPACK 切换 vendored vs 外部库，是多后端矩阵的体现

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| linalg-core 架构图 | `linalg-core-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/linalg-core-architecture.json`
- 降档原因：无
