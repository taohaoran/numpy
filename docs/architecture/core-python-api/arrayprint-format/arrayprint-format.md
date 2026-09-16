# 数组打印与格式化（arrayprint-format）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 数组转字符串 | array2string、array_str、array_repr：将 ndarray 格式化为可读字符串 | `numpy/_core/arrayprint.py:644` |
| 打印配置管理 | set_printoptions、get_printoptions、printoptions（上下文管理器）：全局/局部打印配置 | `arrayprint.py:123,336,398` |
| 浮点格式化 | format_float_positional、format_float_scientific：独立浮点数格式化函数 | `arrayprint.py:1221,1140` |
| 格式化类族 | FloatingFormat、IntegerFormat、BoolFormat、ComplexFloatingFormat、DatetimeFormat、TimedeltaFormat、SubArrayFormat | `arrayprint.py:985,1312,1331,1341,1400,1433,1438` |
| 配置存储 | format_options（ContextVar）：线程/异步隔离的打印配置上下文变量 | `numpy/_core/printoptions.py:31` |
| 区域无关大小写 | english_lower、english_upper、english_capitalize：避免 locale 依赖的 ASCII 大小写转换 | `numpy/_core/_string_helpers.py:16,44,72` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `format_options`（ContextVar） | `printoptions.py:31` | 存储打印配置的线程安全上下文变量，默认值 dict 在 `printoptions.py:14` |
| `FloatingFormat` | `arrayprint.py:985` | 浮点数格式化类，封装 floatmode（fixed/unique/maxprec/maxprec_equal）和 dragon4 算法 |
| `_make_options_dict(...)` | `arrayprint.py:57` | 从关键字参数构造配置 dict，校验 floatmode/sign/legacy/threshold/precision |
| `_get_format_function(data, **options)` | `arrayprint.py:526` | 根据数据 dtype 选择合适的格式化函数（Floating/Integer/Bool/Complex 等） |
| `_formatArray(a, format_function, ...)` | `arrayprint.py:854` | 递归格式化 n 维数组，处理换行、edgeitems 截断、嵌套括号 |
| `printoptions(*args, **kwargs)` | `arrayprint.py:398` | 上下文管理器：进入时设置打印选项，退出时恢复 |

## 3. 关键调用链

### 3.1 print(arr) → 数组字符串化

1. 用户 `print(arr)` 调用 `ndarray.__str__` → C 扩展 → `array2string(arr)`
2. `array2string` 调用 `_array2string(a, options, ...)`（`arrayprint.py:606`）
3. `_array2string` 从 `format_options` ContextVar 读取配置（precision、threshold、edgeitems 等）
4. `_get_format_function(data, **options)` 根据 dtype 选择格式化类（如 FloatingFormat）
5. `_formatArray` 递归遍历数组维度，用格式化函数将每个元素转为字符串，处理 linewidth 换行和 edgeitems 截断
6. 最终返回拼接后的字符串

### 3.2 浮点最短往返表示

1. FloatingFormat 初始化时根据 floatmode 和 precision 配置 dragon4 参数
2. 调用 C 扩展 `dragon4_positional` 或 `dragon4_scientific` 计算最短往返十进制表示
3. dragon4 算法保证 `float(s) == s`（最短往返性质），同时遵守用户的 precision/floatmode 要求

## 4. 配置项

| 配置项 | 默认值 | 位置 |
|--------|--------|------|
| `precision` | 8（位） | `printoptions.py:18` |
| `threshold` | 1000（超过则截断为 ...） | `printoptions.py:16` |
| `edgeitems` | 3（每端显示的元素数） | `printoptions.py:15` |
| `linewidth` | 75 | `printoptions.py:20` |
| `floatmode` | "maxprec" | `printoptions.py:17` |
| `suppress` | False（抑制小指数科学计数） | `printoptions.py:19` |
| `nanstr` / `infstr` | "nan" / "inf" | `printoptions.py:21,22` |
| `sign` | "-" | `printoptions.py:23` |
| `legacy` | sys.maxsize（最新版） | `printoptions.py:27` |

## 5. 错误与重试语义

- **floatmode 校验**：非法值抛出 `ValueError`，允许值为 fixed/unique/maxprec/maxprec_equal（`arrayprint.py:72`）
- **sign 校验**：非法值抛出 `ValueError`，允许值为 ' '、'+'、'-'（`arrayprint.py:76`）
- **threshold 校验**：非数字或 NaN 抛出 TypeError/ValueError（`arrayprint.py:106-110`）
- **legacy 版本映射**：'1.13'→113、'1.21'→121、'1.25'→125、'2.1'→201、'2.2'→202；未知值发 FutureWarning（`arrayprint.py:87-102`）
- **递归保护**：`_recursive_guard` 防止数组元素是自身时无限递归（`arrayprint.py:575`）

## 6. 并发细节

- **ContextVar 隔离**：`format_options` 是 `ContextVar`，每个线程/异步任务有独立的打印配置副本，多线程设置打印选项互不干扰（`printoptions.py:31`）
- **GIL**：dragon4 C 扩展在计算时不释放 GIL（纯 CPU 字符串转换，开销小）
- **线程 ID 获取**：`from _thread import get_ident` 用于递归检测（`arrayprint.py:30`）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/arrayprint.py`：数组格式化引擎
- `numpy/_core/printoptions.py`：ContextVar 配置存储
- `numpy/_core/_string_helpers.py`：区域无关大小写转换

**Out-of-Scope（不在本仓库源码内）**

- `dragon4_positional`/`dragon4_scientific` C 扩展实现——在 `_multiarray_umath` 中
- 标量的 str/repr——在 `scalartypes.c.src` 中（与 arrayprint 不同用途，见 `arrayprint.py:19-23` 注释）

## 8. 与相邻子系统交互

- **上游**：`ndarray.__str__`/`__repr__`（C 扩展）→ `array2string`；用户直接调用 `np.array2string`、`np.set_printoptions`
- **下游 → C 扩展**：`arrayprint.py` → `dragon4_positional`/`dragon4_scientific`（浮点格式化）、`datetime_as_string`（日期时间格式化）
- **下游 → numerictypes**：`arrayprint.py` 使用 `numerictypes.float64/int_/complex128/flexible` 等类型判断

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **能力缝归组**：核心能力缝是 **ContextVar 配置隔离**——`format_options` 使用 `contextvars.ContextVar` 而非模块全局变量，确保多线程和 asyncio 任务中的打印配置互不干扰。这是纯 Python 框架中"配置面工程"的典型案例
2. **格式化策略模式**：`_get_format_function` 根据数据 dtype 选择格式化类（FloatingFormat/IntegerFormat/BoolFormat/ComplexFloatingFormat 等），是策略模式的应用
3. **locale 规避**：`_string_helpers.py` 不使用 `str.lower()`/`str.upper()`（受 locale 影响，如土耳其语 I/i），而是硬编码 ASCII 翻译表，确保类型名称生成的确定性

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| arrayprint-format 架构图 | `arrayprint-format-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/arrayprint-format-architecture.json`
- 降档原因：无，调整 labelDy 后通过 showcase 校验
- 时序图/数据流图：数组格式化是线性递归过程，已在第 3 节文字化描述，不适用额外时序图
