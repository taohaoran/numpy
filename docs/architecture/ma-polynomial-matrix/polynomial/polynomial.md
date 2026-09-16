# polynomial（polynomial）

> 本文是 `ma-polynomial-matrix` 域下的叶子子系统文档。域级总览见 `../ma-polynomial-matrix.md`。
> 本文只展开 NumPy 多项式子包（正交多项式族的抽象基类体系），不重复展开 `masked-array` 与 `matrix`。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 多项式抽象基类 | 定义不可变序列类的算术特例方法与 fit/deriv/integ/roots 等 | `numpy/polynomial/_polybase.py:20`（`ABCPolyBase`） |
| 幂级数族 | Polynomial（普通幂级数） | `numpy/polynomial/polynomial.py:1615` |
| 切比雪夫族 | Chebyshev（第一类） | `numpy/polynomial/chebyshev.py` |
| 勒让德族 | Legendre | `numpy/polynomial/legendre.py` |
| 拉盖尔族 | Laguerre | `numpy/polynomial/laguerre.py` |
| 厄米特族 | Hermite / HermiteE | `numpy/polynomial/hermite.py`、`hermite_e.py` |
| 通用算法 | 与具体基无关的拟合/求根/范德蒙/域映射，按原语参数化 | `numpy/polynomial/polyutils.py` |
| 模块级便捷函数 | polyadd/polysub/polymul/polyval/polyfit/polyder/polyint 等 | 各族模块顶层 |
| 字符串打印 | 上/下标 Unicode 映射、symbol 变量名 | `_polybase.py:80-108` |

对外暴露点：`numpy.polynomial.Polynomial`、`Chebyshev`、`Legendre`、`Laguerre`、`Hermite`、`HermiteE`（经 `numpy/polynomial/__init__.py`）。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `ABCPolyBase(abc.ABC)` | `_polybase.py:20` | 抽象基类；把 `__add__`/`__sub__`/`__mul__`/`__pow__`/`__call__` 等定义为对本族原语的调用 |
| 抽象原语 `_add/_sub/_mul/_div/_pow` | `_polybase.py:131-151` | 各族必须绑定的运算原语（staticmethod） |
| 抽象原语 `_val/_int/_der/_fit` | `:156-171` | 求值/积分/微分/拟合原语 |
| 抽象原语 `_line/_fromroots` | `:176-186` | 线性映射/由根构造原语 |
| `Polynomial` | `polynomial.py:1615` | 幂级数实现；`_add=staticmethod(polyadd)` 等把类属性绑到模块函数（`:1642-1650`） |
| `domain` / `window` / `maxpower` | 各族类属性 | 区间映射与幂次上限（防多项式规模爆炸） |
| `polyutils._fit/_div/_pow/_vander_nd` | `polyutils.py:582/519/670/433` | 与基无关的通用算法，接收各族原语作参数 |
| `mapdomain/getdomain` | `polyutils.py:288/194` | domain→window 线性映射 |

## 3. 关键调用链

**链路一：用户对两个 Polynomial 实例相加**
1. `p1 + p2` 触发 `ABCPolyBase.__add__`（`_polybase.py`），它校验类型/domain 一致。
2. 委托本族绑定的 `_add`（即 `polynomial.py:216` 的 `polyadd(c1, c2)`）。
3. `polyadd` 对系数数组做对齐相加，返回新系数，再由基类包装为同类型新实例。

**链路二：`Chebyshev.fit(x, y, deg)`**
1. 基类 `fit` 类方法收集 x/y/deg/w。
2. 委托本族 `_fit`（= `polyutils._fit`，`polyutils.py:582`）。
3. `_fit` 调本族 `_vander_nd` 构范德蒙矩阵，再用 `_core` 最小二乘解出切比雪夫系数，包装为 `Chebyshev` 实例。

**链路三：求值 `p(x)`**
1. 调用 `__call__` → 本族 `_val`（如 `polyval`，`polynomial.py:663`）。
2. 先把 x 从 domain 映射到 window，再按系数用 Horner 类方案求值。

## 4. 配置项

| 配置 / 类属性 | 默认 / 行为 | 位置 |
|------|------|------|
| `maxpower` | 100，`p**n` 的 n 上限，防规模爆炸 | `_polybase.py:77` |
| `domain` / `window` | 各族默认区间（如 Polynomial 为 `[-1,1]`），实例可覆盖 | 各族类属性 |
| `symbol` | 打印用变量名，默认 `'x'` | `_polybase.py:40` |
| `_use_unicode` | Windows 上默认关 Unicode 上下标 | `_polybase.py:108` |
| `rcond`（fit） | 最小二乘截断比 | `polyutils.py:582` |

## 5. 错误与重试语义

- **domain 不一致**：对不同 domain 的实例做运算时基类自动映射到统一 domain，不报错。
- **幂次超限**：`p**n` 超过 `maxpower` 抛 ValueError。
- **拟合不适定**：最小二乘用 `rcond` 截断；`full=True` 时返回额外诊断信息。
- 无重试/退避；输入错误即抛。

## 6. 并发细节

- 纯 Python 不可变对象（`ABCPolyBase` 实例不可变，`__hash__=None`），GIL 下单线程。
- 实际系数运算下沉 `_core` ufunc；Python 层无锁、无后台线程。
- 实例共享系数数组视图时不可变语义保证并发只读安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/polynomial/_polybase.py`、`polyutils.py`、七个族模块与 `__init__.py`。

**Out-of-Scope（不在本仓库源码内）**
- 最小二乘求解、系数算术的底层在 `_core`（Linalg/ufunc，其他域）。
- 旧版 `numpy.poly1d`（已弃用）与 `numpy.polyfit` 等位于其他模块，不在本新多项式子包。

## 8. 与相邻子系统交互

- **上游**：用户经 `numpy.polynomial` 入口使用；拟合/求值是科学计算常用入口。
- **本叶子 → 下游**：调用 `_core` 数组运算与 Linalg 最小二乘。
- **相邻**：与 `matrix`/`masked-array` 同属"ndarray 的特化类体系"能力缝，但本叶子聚焦正交多项式基。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝（模板方法）**：`ABCPolyBase` 是典型的"抽象基类定义算法骨架、子类填原语"能力缝——算术算子在基类写一次，六个族仅需绑定 `_add/_mul/_fit` 等 staticmethod，新增一族只需补原语。
- **策略/参数化复用**：`polyutils` 的 `_fit/_div/_pow` 接收本族原语函数作参数，实现"同一算法套不同基"。
- **配置驱动静态分派**：行为差异（基函数、domain、符号）由类属性决定，同一套 `ABCPolyBase` 代码经不同子类配置切换，是框架级静态多态。
- **注册表/工厂**：无运行期字符串注册表；族类直接由 `__init__.py` 显式再导出。
- **代码生成/懒加载**：无。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `polynomial-architecture.html` | architecture | showcase |

- 质量档：showcase（0 错误）。初版网格过密触发"边穿无关节点"，将六个族合并为两个节点并改 abc→polyutils 后通过。
- 未补时序/数据流图：模板方法体系是静态类关系，调用链短（第 3 节文字化已足够），补图增量价值低。
- JSON IR 源文件：`json/polynomial-architecture.json`。
