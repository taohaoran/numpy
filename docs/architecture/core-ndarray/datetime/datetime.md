# datetime（datetime）

> 本文是 `core-ndarray` 域下的叶子子系统文档。域级总览见 `../core-ndarray.md`，
> 本文只展开 datetime64/timedelta64 dtype 的核心实现、单位换算、字符串解析与工作日历，
> 不重复展开通用 dtype 描述符机制（见 `../dtype-system/dtype-system.md`）。
>
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| datetime/timedelta dtype 核心 | `datetime.c`：datetime64/timedelta64 类型实现、标量对象、加减/比较、单位元数据 | `numpy/_core/src/multiarray/datetime.c` |
| 分解时间结构 | `npy_datetimestruct`：year/month/day/hour/min/sec/us/ps/as | `numpy/_core/include/numpy/ndarraytypes.h:897` |
| 内部 API 头 | `_datetime.h`：单位换算、can_cast_units、时间结构↔整数转换、hash | `numpy/_core/src/multiarray/_datetime.h` |
| 单位换算 | `can_cast_datetime64_units`/`can_cast_timedelta64_units`：判定单位间安全转换 | `_datetime.h:116`/`:135` |
| ISO8601 解析/格式化 | `datetime_strings.c`：datetime 字符串解析与反向生成 | `numpy/_core/src/multiarray/datetime_strings.c` |
| 工作日函数 | `datetime_busday.c`：`np.is_busday`/`busday_offset`/`busday_count`，weekmask/holidays | `numpy/_core/src/multiarray/datetime_busday.c` |
| 工作日历加速对象 | `datetime_busdaycal.c`：预计算工作日历对象，加速重复工作日计算 | `numpy/_core/src/multiarray/datetime_busdaycal.c` |
| 单位元数据 | `PyArray_DatetimeMetaData`：base unit + num（如 '2D'=2 天） | `ndarraytypes.h:883` |
| NaT | datetime 特殊值 Not-a-Time（year=NPY_DATETIME_NAT） | `_datetime.h`/datetime.c |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `npy_datetimestruct` | `ndarraytypes.h:897` | 分解的日期时间分量 |
| `npy_timedeltastruct` | `ndarraytypes.h:903` | 分解的时间差分分量 |
| `PyArray_DatetimeMetaData` | `ndarraytypes.h:883` | 单位元数据（base + num） |
| `NPY_DATETIMEUNIT` | `_datetime.h` | 单位枚举（Y..as） |
| `can_cast_datetime64_units` | `_datetime.h:116` | 单位间安全转换判定 |
| `convert_datetime_to_pyobject` | `_datetime.h:251` | 整数+元数据 → Python datetime |

## 3. 关键调用链

### 3.1 解析 `np.datetime64('2020-01-01', 'D')`

1. 字符串经 `datetime_strings.c` 解析为 `npy_datetimestruct`。
2. 时间结构经内部换算转为 64 位整数 + 单位（`PyArray_DatetimeMetaData`）。
3. 构造 datetime64 标量；单位间转换走 `can_cast_datetime64_units` 判定 cast 安全。

### 3.2 工作日计算（`np.is_busday(arr)`）

1. `datetime_busday.c` 把每个日期映射到工作日判定。
2. 重复计算前可构造 `datetime_busdaycal.c` 日历对象（预计算 weekmask/holidays）加速。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|---------------|-------------|------|
| `weekmask` | 工作日掩码（默认 Mon-Fri） | datetime_busday.c |
| `holidays` | 节假日数组 | datetime_busday.c |
| `NPY_DATETIMEUNIT` | 单位粒度 | _datetime.h |
| NaT | Not-a-Time 特殊值 | datetime.c |

## 5. 错误与重试语义

- 非法日期/单位字符串解析失败抛 ValueError。
- 不兼容单位转换（如纳秒→年非安全）按 cast 规则拒绝。
- 无重试。

## 6. 并发细节

- datetime 计算为纯函数（时间结构↔整数），无共享可变状态；GIL 保护对象。
- 工作日历对象为不可变（计算后只读），可安全共享。
- 无独立线程/锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- datetime64/timedelta64 dtype 核心、单位换算、ISO 字符串解析
- 工作日函数与日历加速对象

**Out-of-Scope（不在本仓库源码内）**
- 通用 dtype 元类/描述符——见 `dtype-system` 叶子
- 时区库 pytz/dateutil——外部，不在本仓库源码内（仅经 `get_tzoffset_from_pytzinfo` 桥接）
- 实际 ufunc 算术 loop——umath/ 域

## 8. 与相邻子系统交互

- 上游：Python `np.datetime64`/`np.is_busday`；`dtype-system` 注册 datetime64/timedelta64 DType。
- 下游：`datetime_strings.c` 供 dtype 字符串化；`convert_datatype.c` 注册 datetime cast loop；pytz（外部）时区桥接。

## 9. 语言专项适配口径（Python + C 混合）

- **绑定边界**：datetime 核心全在 C（datetime.c/datetime_strings.c/datetime_busday*.c），Python 侧仅薄封装。
- **时间表示**：内部用 64 位整数 + 单位元数据（而非浮点/日历对象），`npy_datetimestruct` 仅在解析/显示时分解。
- **依赖方向**：`_datetime.h` 内部 API → datetime.c → datetime_strings/busday，单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| datetime 架构图 | `datetime-architecture.html` | architecture | standard |
| JSON IR 源 | `json/datetime-architecture.json` | — | — |

降档说明：showcase 因 dt-core→strings 垂直边出侧误判、dt-core→dtype-sys 水平边穿越 dts-struct，改为垂直显式侧并把 dtype 边从 dts-struct 引出后降 standard。时序图不单独补：解析与工作日流程已在第 3 节文字化。
