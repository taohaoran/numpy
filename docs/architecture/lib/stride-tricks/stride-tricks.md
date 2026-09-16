# stride-tricks（stride-tricks）

> 本文是 `lib` 域下的叶子子系统文档。域级总览见 `../lib.md`。
> 本文只展开"按步长/广播规则构造数组视图"的能力，不重复展开 `array-utils` 与 `npyio`。
>
> 源码基准：numpy 主分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 手动步长视图 | 按给定 shape/strides 构造视图，绕过形状-步长一致性检查 | `numpy/lib/_stride_tricks_impl.py:39`（`as_strided`） |
| 滑窗视图 | 在某轴上展开滑动窗口（内部复用 as_strided） | `numpy/lib/_stride_tricks_impl.py:180`（`sliding_window_view`） |
| 广播视图 | 把数组广播到目标形状的只读视图 | `numpy/lib/_stride_tricks_impl.py:475`（`broadcast_to`） |
| 广播形状推导 | 多组 shape 广播结果形状 | `:541`（`broadcast_shapes`）、`:520`（`_broadcast_shape`） |
| 广播数组 | 多数组广播为视图元组 | `:589`（`broadcast_arrays`） |
| 接口字典挂接 | 用占位对象持有 `__array_interface__` 并引用 base | `:15`（`DummyArray`） |
| 子类保留 | 视图若源自子类则尽量保留子类类型 | `:25`（`_maybe_view_as_subclass`） |

薄包装 `numpy/lib/stride_tricks.py` 仅再导出 `as_strided/sliding_window_view`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `DummyArray` | `_stride_tricks_impl.py:15` | 轻量对象，仅承载 `__array_interface__` 字典与 `base` 引用，供 `ndv` 重建视图 |
| `as_strided(x, shape, strides, subok, writeable, check_bounds)` | `:39` | 核心入口；`check_bounds` 可开启越界检查 |
| `sliding_window_view(x, window_shape, axis)` | `:180` | 计算窗口后的新形状与步长，委托 `as_strided` |
| `_broadcast_to / broadcast_to` | `:447/475` | 按广播规则构造只读视图 |
| `_broadcast_shape / broadcast_shapes` | `:520/541` | 纯 Python 推导广播结果形状，不分配内存 |
| `_maybe_view_as_subclass` | `:25` | 若原数组是 ndarray 子类，把新视图 `view(type=...)` 回该子类并触发 `__array_finalize__` |

## 3. 关键调用链

**链路一：`sliding_window_view(x, window_shape, axis)`**
1. 校验 window_shape 维度匹配，计算每个窗口的步长等于原步长、新形状在 window 维展开。
2. 委托 `as_strided`（`_stride_tricks_impl.py:39`）按新 shape/strides 构造视图。
3. `_maybe_view_as_subclass` 保留可能的子类类型。

**链路二：`as_strided` 构造危险视图**
1. 入参缺省时用 `x.shape`/`x.strides`。
2. 通过 `DummyArray`/`__array_interface__` 或直接 `ndarray` 构造共享 `x` 存储的新视图。
3. `writeable` 控制是否可写；`check_bounds` 为 True 时才做越界校验（默认偏危险）。

**链路三：`broadcast_to(array, shape)`**
1. `_broadcast_to`（`:447`）按广播规则把缺失维步长置 0、形状对齐。
2. 产出只读视图，不复制数据。

## 4. 配置项

| 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `writeable` | `as_strided` 默认 True，视图可写（危险） | `:40` |
| `subok` | 是否保留 ndarray 子类类型 | `:40` |
| `check_bounds` | 默认 None；为 True 时检查索引不越界 | `:40` |
| `strict`（broadcast_arrays） | 严格广播与否 | `:589` |

## 5. 错误与重试语义

- **越界读写**：`as_strided` 默认不校验，错误的 strides/shape 会读到无关内存或写坏数据——这是已知的设计性风险，文档以 warning 强调。
- **广播不兼容**：`broadcast_shapes` 对不可广播的形状组合抛 `ValueError`。
- 无重试/退避；参数错误即抛。

## 6. 并发细节

- 纯 Python 视图构造，GIL 下单线程；不创建线程或锁。
- 视图与 base 数组共享内存存储；多线程同时写同一视图由调用方负责同步（本叶子不加锁）。
- `broadcast_to` 视图只读，无并发写风险。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `numpy/lib/_stride_tricks_impl.py`：as_strided / sliding_window_view / broadcast_* / DummyArray。

**Out-of-Scope（不在本仓库源码内）**
- 视图的内存布局与 `ndarray.view()` 底层实现在 `_core`（C 层，其他域）。
- 广播规则的核心判定在 `_core` 的 broadcast 机制；本叶子只做 Python 侧形状推导与视图封装。
- 实际内存字节由 OS/分配器管理。

## 8. 与相邻子系统交互

- **上游**：用户与其他 NumPy 函数；`sliding_window_view` 是信号处理/滚动统计常用入口。
- **本叶子 → 下游**：调用 `_core` 的 ndarray 构造与 `view()`，共享 `base` 数组存储。
- **相邻**：与 `array-utils` 的形状函数（split/tile）同属"视图/形状变换"能力缝，但本叶子聚焦"步长与广播"这一底层机制。

## 9. 语言专项适配口径（纯 Python 项目专项）

- **能力缝**：本叶子是"视图构造原语"能力缝——直接暴露 ndarray 的两步抽象（shape + strides），是其他高级变换（滑动窗口、广播）的底层构建块。
- **懒加载/工厂**：无注册表；`DummyArray` 是轻量适配器，把内存描述字典伪装成类数组对象以喂给 ndarray 构造器。
- **子类型协议**：`_maybe_view_as_subclass` 配合 `__array_finalize__` 是 NumPy 子类协议（ndarray 子类化）的关键钩子体现。
- **代码生成/DRY**：无。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|------|------|------|------|
| 架构图 | `stride-tricks-architecture.html` | architecture | standard |

- 降档原因：节点间纵向引用（sliding→as_strided→DummyArray→core）在 showcase 布局下触发边穿/重叠约束，降 standard；render 退出码 0、HTML 约 805KB。
- 未补时序/数据流图：本叶子仅 4 个入口函数，调用链短（第 3 节文字化已足够），补图增量价值低。
- JSON IR 源文件：`json/stride-tricks-architecture.json`。
