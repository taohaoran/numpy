# f2py（f2py）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 域职责

`numpy.f2py` 是 Fortran 接口代码生成器：把用户的 Fortran 源码解析为 AST，映射为 C/Python 签名，生成 C 包装与 Fortran 包装，再链接 `fortranobject.c` 运行时，最终由外部编译器构建为可 import 的 Python 扩展。本域是 NumPy 与 Fortran 生态之间的桥梁。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 数据流图 | 职责一句话 |
|------|------|--------|----------|-----------|
| f2py-core | [f2py-core.md](f2py-core/f2py-core.md) | [架构图](f2py-core/f2py-core-architecture.html) | [数据流](f2py-core/f2py-core-dataflow.html) | Fortran 解析→C 包装代码生成→运行时 |

## 3. 域级机制细节

- **代码生成管道**：`run_main` → `crackfortran`（AST 字典）→ `capi_maps/rules`（签名映射）→ `buildmodules/cfuncs`（生成 C）→ 外部编译器。
- **生成 vs 手写边界**：`fortranobject.c/.h` 手写入库；`*module.c`、`*-f2pywrappers.f` 每次构建生成、不入库。
- **GIL**：`requires_gil` 选项控制生成包装是否释放 GIL，默认 `Py_MOD_GIL_NOT_USED`。
