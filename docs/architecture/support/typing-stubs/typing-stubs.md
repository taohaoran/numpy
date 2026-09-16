# typing-stubs（typing-stubs）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 `../support.md`。
> 本文只展开 NumPy 的类型注解/类型别名体系，不重复展开其他支撑子系统。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 公共类型别名 | ArrayLike / DTypeLike / NBitBase 等面向用户的类型 | `numpy/typing/__init__.py` |
| 内部类型基件 | `_SupportsArray` 协议、`_Shape`、`_NestedSequence` | `numpy/_typing/_array_like.py`、`_shape.py`、`_nested_sequence.py` |
| 位宽层级 | `_32Bit/_64Bit` 表达整型精度层级 | `numpy/_typing/_nbit_base.py`、`_nbit.py` |
| 标量/dtype 别名 | `_scalars.py`、`_dtype_like.py`、`_char_codes.py` | `numpy/_typing/` |
| 文档字符串 | 给别名补充说明 | `numpy/_typing/_add_docstring.py` |
| 测试 | 类型标注的运行时/静态测试 | `numpy/typing/tests/`（test_typing/test_runtime） |

对外暴露点：`numpy.typing.ArrayLike`、`numpy.typing.DTypeLike`。

## 2. 核心类型与接口清单

| 类型 / 别名 | 位置 | 职责 |
|------|------|------|
| `ArrayLike` | `numpy/typing/__init__.py` | "可被 `np.array` 接受"的联合类型 |
| `DTypeLike` | `numpy/typing/__init__.py` | 可转为 dtype 的输入类型 |
| `_SupportsArray[DTypeT]` | `_typing/_array_like.py:24` | `@runtime_checkable` Protocol，刻画 `__array__` 鸭子类型 |
| `NDArray[ScalarT]` | `_array_like.py:22` | PEP 695 泛型 ndarray 类型别名 |
| `_32Bit/_64Bit` | `_typing/_nbit_base.py` | 整型位宽类型层级（用于 uintp/intp 推断） |

## 3. 关键调用链

1. 用户 `import numpy.typing as npt` 标注 `arr: npt.ArrayLike`。
2. 静态检查器（mypy/pyright）解析 `numpy/typing/__init__.py` 再导出的别名。
3. 别名最终落到 `_SupportsArray` 等 Protocol 与 ndarray 泛型，检查 `__array__` 实现是否匹配。

## 4. 配置项

- 无运行期 flags；本叶子是类型定义，不影响运行时行为。
- 类型严格度在 `numpy/typing/__init__.py` 文档中说明（比运行时 API 更严格）。

## 5. 错误与重试语义

- 类型错误由外部检查器在静态期报告；本叶子运行期不抛错。
- 无重试。

## 6. 并发细节

- 纯类型定义，无运行逻辑、无线程/锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/_typing/`、`numpy/typing/`（含 tests）。

**Out-of-Scope（不在本仓库源码内）**
- mypy/pyright 等类型检查器本体——外部。
- 历史 mypy 插件已迁移；现以标准 type-annotated 包 + 测试表达。

## 8. 与相邻子系统交互

- **上游**：用户与下游库的类型标注。
- **本叶子 → 下游**：引用 `numpy.ndarray`、`numpy.dtype`（`_core`）作泛型参数。
- 与 `_core` 的 `.pyi` stub 配合完成完整类型覆盖。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝（Protocol/鸭子类型）**：`_SupportsArray` 是 `__array__` 协议的 `@runtime_checkable` Protocol，刻画"能转成数组"的第三方类型，是 NumPy 类型系统的扩展缝。
- **类型别名工厂**：ArrayLike/DTypeLike 是联合类型别名，组合多个基本类型。
- **懒加载/TYPE_CHECKING**：`_array_like.py` 用 `if TYPE_CHECKING` 条件导入避免运行期重依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `typing-stubs-architecture.html` | architecture | standard |

- 降档原因：pub→proto 竖边标签重叠，`labelDy` 补偿后落 standard；render 退出码 0、HTML 约 803KB。
- 未补时序/数据流图：纯类型定义，无运行期数据流。
- JSON IR 源文件：`json/typing-stubs-architecture.json`。
