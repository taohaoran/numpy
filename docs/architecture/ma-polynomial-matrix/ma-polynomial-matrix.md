# ma-polynomial-matrix（ma-polynomial-matrix）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域聚集三类"ndarray 的特化类体系"：掩码数组（`numpy.ma`，带掩码与填充值）、多项式子包（`numpy.polynomial`，正交多项式族的模板方法体系）、2-D 矩阵（`numpy.matrixlib`，已不推荐但仍维护）。三者都通过继承/组合 ndarray 复用存储与索引，再叠加各自的语义层。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|-----------|
| masked-array | [masked-array.md](masked-array/masked-array.md) | [架构图](masked-array/masked-array-architecture.html) | 带掩码与 fill_value 的 ndarray 子类 |
| polynomial | [polynomial.md](polynomial/polynomial.md) | [架构图](polynomial/polynomial-architecture.html) | 正交多项式族的 ABCPolyBase 模板方法体系 |
| matrix | [matrix.md](matrix/matrix.md) | [架构图](matrix/matrix-architecture.html) | 2-D matrix 子类（`*` 为矩阵乘） |

## 3. 域级机制细节

- **ndarray 子类化**：三者均继承 ndarray；`__array_priority__`（matrix）、`__array_finalize__`（masked-array）等钩子控制类型提升。
- **模板方法（polynomial）**：ABCPolyBase 定义算法骨架，六族仅绑定 `_add/_mul/_fit` 等 staticmethod 原语。
- **ufunc 包装（masked-array）**：`_MaskedUnaryOperation` 等把掩码传播织进每个运算。
