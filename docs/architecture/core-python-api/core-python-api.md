# Python 前端域（core-python-api）域总览

> 本域包含 NumPy 核心 Python API 层的 6 个叶子子系统；各叶子详情见对应文档。
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

本域是 NumPy 的 Python 前端层——用户直接接触的 `numpy._core/` 模块。NumPy 2.x 将 `numpy/core/` 拆分为 `numpy/_core/`（真实实现）和 `numpy/core/`（向后兼容再导出薄层）。本域覆盖 ndarray 的创建、操作、打印、记录数组、覆盖协议和类型系统的 Python 包装层，按「纯 Python 项目专项分析清单」分析：能力缝归组、注册表与工厂、懒加载、配置驱动分派、`__array_function__`/`__array_ufunc__` 覆盖协议。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|------------|
| numeric-layer | [numeric-layer.md](numeric-layer/numeric-layer.md) | [架构图](numeric-layer/numeric-layer-architecture.html) | ndarray 创建/操作的核心函数层（numeric.py、fromnumeric.py 等） |
| function-base | [function-base.md](function-base/function-base.md) | [架构图](function-base/function-base-architecture.html) | 通用函数与形状操作（shape_base、einsumfunc） |
| arrayprint-format | [arrayprint-format.md](arrayprint-format/arrayprint-format.md) | [架构图](arrayprint-format/arrayprint-format-architecture.html) | 数组打印格式化与 printoptions 配置 |
| records-strings | [records-strings.md](records-strings/records-strings.md) | [架构图](records-strings/records-strings-architecture.html) | 记录数组与字符数组（records、defchararray） |
| overrides-ufunc-config | [overrides-ufunc-config.md](overrides-ufunc-config/overrides-ufunc-config.md) | [架构图](overrides-ufunc-config/overrides-ufunc-config-architecture.html) | NEP-18 `__array_function__`/`__array_ufunc__` 覆盖协议 |
| numerictypes-getlimits | [numerictypes-getlimits.md](numerictypes-getlimits/numerictypes-getlimits.md) | [架构图](numerictypes-getlimits/numerictypes-getlimits-architecture.html) | 数值类型系统与 finfo/iinfo 工具 |

## 3. 域级机制细节

- **NEP-18 覆盖协议**：`overrides.array_function_dispatch` 装饰器 + `_ArrayFunctionDispatcher`（C 扩展）实现 duck typing 能力缝——第三方数组库（如 Dask、CuPy）可通过 `__array_function__` 覆盖 NumPy 函数
- **`_NoValue` 单例**：禁止 reload 保身份，用于区分"未传参"与"None"
- **numpy/core/ 薄层**：NumPy 2.x 的 `numpy/core/` 是 `numpy/_core/` 的再导出，保持向后兼容
- **printoptions 上下文管理**：`np.printoptions()` 上下文管理器临时修改打印格式

## 4. 域级图

本域各叶子独立出图，无域级架构图。
