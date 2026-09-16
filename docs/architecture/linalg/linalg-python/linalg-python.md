# 线性代数 Python API（linalg-python）

> 本文是 `linalg` 域下的叶子子系统文档。域级总览见 `../linalg.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 矩阵求解 | solve（线性方程组）、tensorsolve、tensorinv | `numpy/linalg/_linalg.py:374,293,472` |
| 矩阵求逆/幂 | inv、matrix_power | `_linalg.py:547,667` |
| 特征值分解 | eig、eigvals、eigh、eigvalsh | `_linalg.py` |
| 奇异值分解 | svd、svdvals | `_linalg.py` |
| 其他分解 | cholesky、qr、lstsq、det、slogdet | `_linalg.py` |
| 范数/秩 | norm、matrix_rank、cond | `_linalg.py` |
| 结果 NamedTuple | EigResult、EighResult、QRResult、SlogdetResult、SVDResult | `_linalg.py:82-100` |
| 错误处理 | _raise_linalgerror_singular/nonposdef/eigenvalues_nonconvergence/svd_nonconvergence/lstsq/qr | `_linalg.py:142-157` |
| 类型提升 | _commonType、_realType、_complexType、isComplexType | `_linalg.py:167-200` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `solve(a, b)` | `_linalg.py:374` | 求解线性方程 a @ x = b |
| `inv(a)` | `_linalg.py:547` | 矩阵求逆 |
| `eig(a)` | `_linalg.py` | 特征值/特征向量分解（非对称） |
| `svd(a)` | `_linalg.py` | 奇异值分解 |
| `cholesky(a)` | `_linalg.py` | Cholesky 分解 |
| `QRResult`/`SVDResult` 等 NamedTuple | `_linalg.py:90-100` | 结构化分解结果 |
| `_commonType(*arrays)` | `_linalg.py:200` | 数组类型提升规则 |
| `LinAlgError` | 继承自 numpy.linalg | 线性代数错误基类 |

## 3. 关键调用链

### 3.1 np.linalg.solve(a, b) 调用链

1. 用户调用 `np.linalg.solve(a, b)` → `_linalg.solve`（经 NEP-18 dispatcher）
2. `solve()` 调用 `_makearray(a)` 将输入转为 2D 数组
3. `_commonType(a, b)` 确定计算精度（float32→float64 提升）
4. 调用 `_umath_linalg.solve(a, b, signature=...)`（C++ ufunc）
5. C++ 层调用 LAPACK dgesv/zgesv 执行求解
6. 返回解数组 x

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| float32 提升 | float32 输入自动提升为 float64（LAPACK 精度） | `_commonType` |
| 外部线程控制 | 依赖 threadpoolctl 控制外部 BLAS 线程数 | `__init__.py` |

## 5. 错误与重试语义

- **奇异矩阵**：solve/inv 遇到奇异矩阵时抛出 LinAlgError（_raise_linalgerror_singular）
- **非正定矩阵**：cholesky 遇到非正定矩阵时抛出 LinAlgError
- **特征值不收敛**：eig/eigh 不收敛时抛出 LinAlgError
- **SVD 不收敛**：svd 不收敛时抛出 LinAlgError
- **lstsq 错误**：lstsq 失败时抛出 LinAlgError
- **无重试机制**：LAPACK 调用失败直接抛出异常

## 6. 并发细节

- Python 层不管理线程；LAPACK 计算在 C++ 层释放 GIL
- 外部 BLAS 库（OpenBLAS/MKL）通常多线程，由 threadpoolctl 控制线程数

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/linalg/_linalg.py`：Python API
- `numpy/linalg/__init__.py`：再导出

**Out-of-Scope（不在本项目原创源码内）**

- `_umath_linalg` C++ 扩展——见 `linalg-core` 叶子
- 外部 BLAS/LAPACK 库——不在本仓库源码内

## 8. 与相邻子系统交互

- **上游**：`numpy/__init__.py` 从 `numpy.linalg` 再导出；用户通过 `np.linalg.*` 调用
- **下游 → linalg-core**：`_linalg.py` → `_umath_linalg`（C++ ufunc）→ LAPACK

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **配置驱动静态分派**：同一 solve 函数根据输入 dtype 分派到 float32/float64/complex64/complex128 四种 LAPACK 路径（由 _commonType 决定）
2. **结构化结果**：使用 NamedTuple 提供类型化的分解结果（EigResult/SVDResult 等），这是 Python 3.9+ 的现代 API 设计
3. **NEP-18 覆盖协议**：所有公开函数经 `@array_function_dispatch` 包装，支持 duck array 覆盖

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| linalg-python 架构图 | `linalg-python-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/linalg-python-architecture.json`
- 降档原因：无
