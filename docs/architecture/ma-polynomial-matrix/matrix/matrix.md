# matrix（matrix）

> 本文是 `ma-polynomial-matrix` 域下的叶子子系统文档。域级总览见 `../ma-polynomial-matrix.md`。
> 本文只展开 `numpy.matrixlib` 的 2-D 矩阵子类，不重复展开 `masked-array` 与 `polynomial`。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 矩阵类 | ndarray 的 2-D 子类，`*` 重载为矩阵乘法 | `numpy/matrixlib/defmatrix.py:74`（`matrix`） |
| 矩阵工厂 | `asmatrix` 不复制地把数据转成 matrix | `defmatrix.py:37` |
| 分块拼接 | `bmat` 按字符串/嵌套列表组装分块矩阵 | `defmatrix.py:1041` |
| 字符串构造 | 解析 `"1,2;3,4"` 形式字符串为矩阵 | `defmatrix.py:16`（`_convert_from_string`） |
| 便捷属性 | `.T` 转置、`.I` 求逆、`.A` 转 ndarray、`.H` 共轭转置 | `defmatrix.py:974`（H）等 |

对外暴露点：`numpy.matrix`、`numpy.asmatrix`、`numpy.bmat`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `matrix(N.ndarray)` | `defmatrix.py:74` | 2-D 矩阵子类 |
| `__array_priority__ = 10.0` | `defmatrix.py:117` | 与 ndarray 混合运算时让结果优先为 matrix 类型 |
| `__new__` | `defmatrix.py:119` | 接受 data/dtype/copy；字符串输入走 `_convert_from_string` |
| `.H` / `.T` / `.I` / `.A` | `:974` 等 | 共轭转置/转置/逆/转 ndarray |
| `asmatrix` | `:37` | 视图式转换工厂 |
| `bmat` | `:1041` | 分块矩阵组装 |

## 3. 关键调用链

**链路一：构造矩阵**
1. `np.matrix("1 2; 3 4")` → `__new__`（`defmatrix.py:119`）识别字符串。
2. 调 `_convert_from_string`（`:16`）解析为 2-D 数组，包装为 matrix。

**链路二：矩阵乘法**
1. `A * B` 因 matrix 重载 `__mul__` 为矩阵乘法（`@`/`matmul` 语义），区别于 ndarray 的逐元素乘。
2. 结果按 `__array_priority__=10` 保持 matrix 类型。

**链路三：`bmat([[a, b], [c, d]])`**
1. `bmat`（`:1041`）把各块沿轴拼接为整体矩阵。

## 4. 配置项

| 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `copy`（matrix） | 默认 True；asmatrix 则不复制 | `:119` |
| `dtype` | 可指定元素类型 | `:119` |
| `__array_priority__` | 10.0，类型提升优先级 | `:117` |

## 5. 错误与重试语义

- **非 2-D**：matrix 始终强制 2-D（1-D 被压成行/列矩阵）。
- **形状不匹配的乘法**：由底层 matmul 抛 `ValueError`。
- 无重试；错误即抛。

## 6. 并发细节

- ndarray 子类，计算下沉 `_core` ufunc；Python 层无锁无线程。
- matrix 与 ndarray 共享存储视图，并发安全由调用方保证。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/matrixlib/defmatrix.py`、`__init__.py`。

**Out-of-Scope（不在本仓库源码内）**
- 矩阵乘法/求逆的底层在 `_core`/linalg（其他域）。
- 官方文档已建议新代码优先用 `ndarray` + `@` 运算符，matrix 处于维护模式。

## 8. 与相邻子系统交互

- **上游**：`numpy/__init__.py` 导出 matrix/asmatrix/bmat。
- **本叶子 → 下游**：继承 ndarray，复用 `_core` 全部运算。
- **相邻**：与 masked-array/polynomial 同属"ndarray 特化子类"能力缝。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝（ndarray 子类化）**：matrix 是 ndarray 子类化的典型样例——通过重载 `__mul__` 与设置 `__array_priority__` 改变运算语义与类型提升方向。
- **工厂**：`asmatrix` 视图式转换、`bmat` 组装式构造是两类工厂缝。
- 本叶子是"已不推荐新代码使用"的历史兼容类，功能面窄。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `matrix-architecture.html` | architecture | showcase |

- 质量档：showcase（0 错误）。
- 未补时序/数据流图：本叶子仅一个类 + 两个工厂，调用链短，补图增量价值低。
- JSON IR 源文件：`json/matrix-architecture.json`。
