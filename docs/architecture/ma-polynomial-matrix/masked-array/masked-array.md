# masked-array（masked-array）

> 本文是 `ma-polynomial-matrix` 域下的叶子子系统文档。域级总览见 `../ma-polynomial-matrix.md`。
> 本文只展开掩码数组 `numpy.ma`，不重复展开 `polynomial` 与 `matrix`。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 掩码数组类 | 继承 ndarray，额外存 `_mask` 与 `fill_value` | `numpy/ma/core.py:2771`（`MaskedArray`） |
| 掩码构造 | masked_where/masked_greater/masked_values/masked_invalid 等 | `numpy/ma/core.py:1886-2401` |
| 掩码工具 | make_mask/getmask/getmaskarray/mask_or | `core.py:1608/1412/1475/1760` |
| 填充值体系 | default/minimum/maximum_fill_value、filled、set/get_fill_value | `core.py:261/332/384/632/521/586` |
| 掩码 ufunc 包装 | 把原生 ufunc 包装成传播掩码的运算 | `core.py:945`（`_MaskedUFunc` 族） |
| 定义域运算 | 对定义域（如 log 自变量>0）额外掩越界 | `core.py:849-929`、`:1176`（`_DomainedBinaryOperation`） |
| ma 统计/集合 | average/median/unique/intersect1d/isin 等掩码版本 | `numpy/ma/extras.py` |
| 掩码记录数组 | 结构化 dtype 的掩码记录子类 | `numpy/ma/mrecords.py:76`（`MaskedRecords`） |
| 记录构造 | fromarrays/fromrecords/fromtextfile/addfield | `mrecords.py:471/514/636/705` |

对外暴露点：`numpy.ma.array`、`MaskedArray`、`ma.masked_where`、`ma.masked_invalid` 等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `MaskedArray(ndarray)` | `core.py:2771` | 掩码数组；`data`/`mask`/`fill_value` 三要素 |
| `MAError / MaskError` | `core.py:143/151` | 掩码操作错误异常族 |
| `_MaskedUFunc` | `core.py:945` | 掩码 ufunc 包装基类 |
| `_MaskedUnaryOperation` | `core.py:956` | 一元 ufunc 包装（如 sin），结果掩码=输入掩码 |
| `_MaskedBinaryOperation` | `core.py:1030` | 二元 ufunc 包装（如 add），结果掩码=输入掩码并集 |
| `_DomainedBinaryOperation` | `core.py:1176` | 带定义域检查的二元运算（如 true_divide、log） |
| `default/minimum/maximum_fill_value` | `core.py:261/332/384` | 按 dtype 推断默认/极值填充值 |
| `filled(a, fill_value)` | `core.py:632` | 把被掩元素替换为 fill_value，返回普通 ndarray |
| `make_mask / mask_or` | `core.py:1608/1760` | 构造/合并掩码 |
| `MaskedRecords(MaskedArray)` | `mrecords.py:76` | 结构化记录的掩码子类，逐字段掩码 |

## 3. 关键调用链

**链路一：两个 MaskedArray 相加**
1. `a + b` 触发 `MaskedArray` 上以 `_MaskedBinaryOperation(np.add)` 绑定的运算。
2. 包装器对 `a._data`、`b._data` 调原生 `np.add` 得数据结果。
3. 按输入掩码求并集（`mask_or`）得到结果掩码，`fill_value` 按规则合并。
4. 返回新 `MaskedArray`（`masked_result._mask = m`，`core.py:1103`）。

**链路二：`filled()` 去掩码**
1. `arr.filled()`（或 `filled(arr)`，`core.py:632`）遍历。
2. 被掩位置替换为 `fill_value`（`_recursive_filled`，`core.py:2546`）。
3. 返回普通 ndarray（不再带掩码）。

**链路三：`ma.masked_invalid(a)`**
1. 识别 NaN/inf（`core.py:2401`）。
2. 把这些位置写入新掩码，返回 MaskedArray。

**链路四：`ma.average`（extras）**
1. `extras.py:537` 先 `filled` 或用掩码权重。
2. 仅对未掩元素加权求和，掩码元素不计入。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `fill_value` | 每实例一个；缺省按 dtype 推断（`default_fill_value`） | `core.py:261` |
| `shrink` | 是否收缩全 False 掩码为 `nomask` | `core.py:1608` |
| `copy` | masked_*/filled 是否复制数据 | 各构造函数 |
| `hard_mask` | 硬掩码模式下不可再被运算"软化" | `MaskedArray` 属性 |
| `rtol/atol`（masked_values） | 浮点相等容差 | `core.py:2328` |

## 5. 错误与重试语义

- **掩码与数据形状不一致**：`make_mask`/赋值时校验，不符抛 `MAError`。
- **填充值类型不符**：`_check_fill_value`（`core.py:468`）按 dtype 转换/警告。
- **全掩归约**：`ma.mean` 等对全掩切片返回 fill_value 或 NaN，并可告警。
- 无重试/退避；参数错误即抛。

## 6. 并发细节

- MaskedArray 是 ndarray 子类，计算下沉 `_core` ufunc（释放 GIL）；Python 层无锁。
- 掩码合并（`mask_or`）是同步数组运算，GIL 下单线程调度。
- 多线程写同一 MaskedArray 需调用方同步（本叶子不加锁）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/ma/core.py`、`extras.py`、`mrecords.py`、`testutils.py`。

**Out-of-Scope（不在本仓库源码内）**
- 原生 ufunc 实现位于 `_core`（C 层，其他域）；本叶子只做掩码传播包装。
- 旧 `numpy.ma` 的 C 加速路径已移除，现纯 Python 包装。

## 8. 与相邻子系统交互

- **上游**：用户经 `numpy.ma`；`numpy.genfromtxt` 可输出 MaskedArray（经 `npyio`）。
- **本叶子 → 下游**：大量调用 `_core` ndarray 与原生 ufunc。
- **相邻**：`extras` 把 `array-utils` 的统计/集合函数做掩码版包装；`mrecords` 面向结构化 dtype。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝（ndarray 子类化 + ufunc 包装）**：MaskedArray 继承 ndarray 复用全部存储/索引机制，再用 `_MaskedUFunc` 族把"掩码传播"织进每个 ufunc 运算——这是组合式扩展缝，而非修改 ndarray 本身。
- **工厂/默认值推断**：`default/minimum/maximum_fill_value` 按 dtype 工厂式推断填充值，体现"类型驱动默认配置"。
- **双轨**：`filled()` 提供"掩码数组 ↔ 普通数组"的双向转换缝。
- **代码生成/懒加载**：无。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `masked-array-architecture.html` | architecture | standard |

- 降档原因：ma→mask 竖边标签与 ma 节点重叠，`labelDy` 补偿后仍落 standard；render 退出码 0、HTML 约 808KB。
- 未补时序图：掩码传播逻辑已在第 3 节文字化（数据+掩码+fill_value 三要素同步演进），补图增量价值低。
- JSON IR 源文件：`json/masked-array-architecture.json`。
