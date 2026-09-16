# lib（lib）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

`numpy/lib` 是 NumPy 的"工具箱"：承载磁盘 IO（.npy/.npz/文本）、公共数组辅助函数族（集合运算、统计、NaN 感知、形状变换、类型判定）以及视图/广播技巧。实现自 NumPy 1.x 起逐步从 `numpy/lib/xxx.py` 迁移到 `_xxx_impl.py`，原文件变为再导出薄包装。本域代码多为"薄编排层"，真正的计算下沉 `_core` ufunc。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| npyio | [npyio.md](npyio/npyio.md) | [架构图](npyio/npyio-architecture.html) | [数据流](npyio/npyio-dataflow.html) | .npy/.npz 二进制格式与文本 IO |
| array-utils | [array-utils.md](array-utils/array-utils.md) | [架构图](array-utils/array-utils-architecture.html) | — | 集合运算/统计/NaN/形状/类型辅助函数族 |
| stride-tricks | [stride-tricks.md](stride-tricks/stride-tricks.md) | [架构图](stride-tricks/stride-tricks-architecture.html) | — | 按步长/广播规则构造数组视图 |

## 3. 域级机制细节

- **薄包装/`_impl` 拆分**：`npyio.py`、`format.py`、`array_utils.py`、`stride_tricks.py` 仅 `from ._xxx_impl import ...`，实现集中在 `_*_impl.py`，配合 `@set_module` 维持对外模块名。
- **`__array_function__` 协议**：array-utils 中每个公开函数配 `_xxx_dispatcher`，供第三方数组库拦截。
- **GIL 边界**：本域 Python 编排层无锁无线程；计算下沉 `_core` ufunc 释放 GIL。
