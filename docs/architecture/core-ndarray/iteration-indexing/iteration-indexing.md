# iteration-indexing（iteration-indexing）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开数组的迭代器对象（逐元素/多数组/邻域）、序列协议与花哨索引（item selection），
> 不重复展开 ndarray 对象本身（见 `../ndarray-object/ndarray-object.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。
> 说明：任务描述中的 `indexed_iterators/` 目录在本 commit 不存在；相关索引逻辑实现在 `item_selection.c`，现代迭代器在 `nditer_*.c`（属相邻文件，本叶子仅引用）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 逐元素迭代器类型 | `PyArrayIter_Type`：把任意 ndarray 展平为一维逐元素迭代器 | `numpy/_core/src/multiarray/iterators.c:1096` |
| 多数组广播迭代器 | `PyArrayMultiIter_Type`：对多个可广播数组同步迭代 | `iterators.c:1520` |
| 邻域迭代器 | `PyArrayNeighborhoodIter_Type`：窗口/邻域遍历（常用于通用 ufunc 边界） | `iterators.c:1803` |
| 迭代器创建/步进 | `NpyIter`-风格辅助：迭代器 new/next/reset 内部函数 | `iterators.c:109`/`:144`/`:220` |
| 序列协议 | `array_as_sequence`：sq_length/sq_item/sq_ass_item/sq_contains；sq_concat 显式报错引导 `np.concatenate` | `numpy/_core/src/multiarray/sequence.c:68` |
| 成员判断 | `array_contains`：`el in arr` 等价于 `(arr==el).any()` | `sequence.c:31` |
| 花哨索引 | `item_selection.c`：整数数组/布尔掩码/元组索引的选元素与赋值 | `numpy/_core/src/multiarray/item_selection.c` |
| 0-d 数组迭代禁止 | `array_iter`：0-d 数组迭代抛 TypeError | `arrayobject.c:1257` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `PyArrayIter_Type` | `iterators.c:1096` | 逐元素迭代器 Python 类型 |
| `PyArrayMultiIter_Type` | `iterators.c:1520` | 广播多迭代器类型 |
| `PyArrayNeighborhoodIter_Type` | `iterators.c:1803` | 邻域迭代器类型 |
| `array_as_sequence` | `sequence.c:68` | 序列协议方法表 |
| `array_contains` | `sequence.c:31` | 成员判断实现 |
| `item_selection.c` 选元素函数 | `item_selection.c` | 花哨索引核心 |

## 3. 关键调用链

### 3.1 for 循环迭代数组（`for x in arr`）

1. Python 调 `array_iter`（`arrayobject.c:1257`）：0-d 抛 TypeError；否则返回 `PySeqIter_New((PyObject*)arr)`。
2. 实际逐项经序列协议 `sq_item`（`array_item`）→ `item_selection.c` 取出第 i 个元素。
3. 多数组广播用 `PyArrayMultiIter_Type`（`iterators.c:1520`）同步步进。

### 3.2 成员判断（`el in arr`）

1. `array_contains`（`sequence.c:31`）调 `PyObject_RichCompare(arr, el, Py_EQ)`。
2. 结果经 `PyArray_Any(..., NPY_RAVEL_AXIS, NULL)` 归约为标量布尔。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| NPY_RAVEL_AXIS | Any 轴归约 | sequence.c |
| 迭代顺序 | C/F 序（由 strides 决定） | iterators.c |
| `np.nditer` 现代迭代器 flags | 见 nditer_constr.c | 相邻文件 |

## 5. 错误与重试语义

- 0-d 数组迭代：`array_iter` 抛 TypeError（`arrayobject.c:1260`）。
- 序列拼接：`array_concat` 抛 TypeError 引导 `np.concatenate`（`sequence.c:54`）。
- 索引越界/形状不匹配：item_selection.c 抛 IndexError。
- 无重试。

## 6. 并发细节

- 迭代器为 Python 迭代器对象，GIL 保护；无独立线程。
- 迭代器内部维护索引指针（stride 步进），单线程消费；多线程并发迭代同一数组外部同步。
- 无锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 三代迭代器类型（逐元素/多数组/邻域）
- 序列协议、成员判断、花哨索引

**Out-of-Scope（不在本仓库源码内）**
- NpyIter 现代通用广播迭代器（`nditer_constr.c`/`nditer_api.c`/`nditer_pywrap.c`）——相邻文件/叶子
- 实际元素访问的 stride loop——`lowlevel_strided_loops.c.src`
- 排序/搜索——见 `core-sort/sort-algorithms` 叶子

## 8. 与相邻子系统交互

- 上游：Python for-loop、`arr[i]` 索引、`np.ndenumerate` 等。
- 下游：ndarray-object（被迭代数据）；item_selection → dtype_transfer（选后赋值转换）；NpyIter（现代迭代路径）。

## 9. 语言专项适配口径（Python + C 混合）

- **绑定边界**：迭代器对象全在 C 侧（iterators.c/sequence.c/item_selection.c），Python 经协议表访问。
- **生成 vs 手写**：本叶子无 `.src` 模板（item_selection.c/iterators.c 为手写 C）；`lowlevel_strided_loops.c.src` 是相邻模板文件。
- **三代迭代并存**：旧式 `PyArrayIter`（逐元素）、`PyArrayMultiIter`（广播）、`NpyIter`（现代通用，nditer_* 文件）——新旧并存，新代码推荐 NpyIter。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| iteration-indexing 架构图 | `iteration-indexing-architecture.html` | architecture | standard |
| JSON IR 源 | `json/iteration-indexing-architecture.json` | — | — |

降档说明：showcase 因垂直边与标签 clearance 约束未全过，降 standard 渲染（已为垂直边显式 fromSide/toSide、标签 labelDy 偏移）。时序图不单独补：迭代与成员判断流程已在第 3 节文字化。
