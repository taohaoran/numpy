# 覆盖协议与 ufunc 配置（overrides-ufunc-config）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| NEP-18 覆盖协议装饰器 | `array_function_dispatch`：将公开函数包装为 `_ArrayFunctionDispatcher`，支持第三方数组库通过 `__array_function__` 接管实现 | `numpy/_core/overrides.py:119` |
| 分发器 C 类 | `_ArrayFunctionDispatcher`：C 扩展实现的分发器，调用时先检查参数的 `__array_function__` 方法 | `overrides.py:7-9`（从 `_multiarray_umath` 导入） |
| 实现参数收集 | `_get_implementing_args`：收集所有有 `__array_function__` 方法的参数，按正确顺序排序 | `overrides.py:76-91` |
| 签名校验 | `verify_matching_signatures`：开发期检查 dispatcher 与实现函数签名一致 | `overrides.py:97` |
| 文档模板替换 | `finalize_array_function_like`：将函数文档中的 `${ARRAY_FUNCTION_LIKE}` 替换为 `like=` 参数说明 | `overrides.py:39` |
| 已注册函数集合 | `ARRAY_FUNCTIONS`：set 类型，记录所有被 `@array_function_dispatch` 包装的公开函数 | `overrides.py:15,35,188` |
| 规约快速路径 | `_ReductionKind`：枚举（SUM_PROD/MIN_MAX/ANY_ALL），供 C 层精确 ndarray 规约快速路径使用 | `overrides.py:18-22` |
| ufunc 错误配置 | seterr/geterr：浮点错误处理策略（divide/over/under/invalid → ignore/warn/raise/call/print/log） | `numpy/_core/_ufunc_config.py:20,112` |
| ufunc 缓冲区大小 | setbufsize/getbufsize：控制 ufunc 内部临时缓冲区大小（默认 8192 字节） | `_ufunc_config.py:162,208` |
| 自定义错误回调 | seterrcall/geterrcall：注册 Python 函数作为浮点错误回调 | `_ufunc_config.py:234,329` |
| 错误状态上下文 | errstate：上下文管理器，临时修改错误处理策略，退出时恢复 | `_ufunc_config.py:387` |
| 全局单例 | `_NoValue`：哨兵单例，区分"未传参"与"显式 None"；`_CopyMode`：枚举（ALWAYS/NEVER/IF_NEEDED） | `numpy/_globals.py` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `array_function_dispatch(dispatcher, module, verify, docs_from_dispatcher, reduction)` | `overrides.py:119` | 装饰器工厂，返回 decorator 将 implementation 包装为 `_ArrayFunctionDispatcher` |
| `_ArrayFunctionDispatcher`（C 类） | `_multiarray_umath` → `overrides.py:8` | 分发器：调用时先检查 `__array_function__`，再路由到实现或第三方 |
| `_get_implementing_args(relevant_args)` | `_multiarray_umath` → `overrides.py:8` | 收集有 `__array_function__` 的参数并排序 |
| `verify_matching_signatures(implementation, dispatcher)` | `overrides.py:97` | 签名一致性校验 |
| `_ReductionKind(enum.IntEnum)` | `overrides.py:18` | 规约类型枚举，与 C 层 `arrayfunction_override.c` 保持同步 |
| `errstate` | `_ufunc_config.py:387` | 上下文管理器，临时修改浮点错误处理 |
| `_NoValueType` / `_NoValue` | `_globals.py` | 单例哨兵值 |
| `_CopyMode(enum.Enum)` | `_globals.py` | 拷贝模式枚举 |

## 3. 关键调用链

### 3.1 NEP-18 分发调用链（np.sum）

1. 用户调用 `np.sum(a)` → `sum` 是 `_ArrayFunctionDispatcher` 实例
2. C 层 `_ArrayFunctionDispatcher.__call__` 调用 dispatcher 函数 `_sum_dispatcher(a, ...)` 获取相关参数 `(a,)`
3. `_get_implementing_args((a,))` 检查 `a` 是否有 `__array_function__` 方法
4. 若 `a` 是第三方数组（如 torch tensor），调用 `a.__array_function__(np.sum, types, args, kwargs)` 委托第三方实现
5. 若无覆盖（纯 ndarray），调用原始 implementation 函数 `sum(...)`
6. 最终走 fromnumeric.py 的 `_wrapreduction` 路径

### 3.2 errstate 上下文

1. 用户 `with np.errstate(divide='ignore'):` → 进入 `errstate.__enter__`
2. 保存当前错误设置，设置新的 divide='ignore'
3. 块内 ufunc 除零错误被忽略
4. `__exit__` 恢复原始错误设置

## 4. 配置项

| 配置项 | 默认值 | 位置 |
|--------|--------|------|
| `divide`（浮点错误） | 'warn' | `_ufunc_config.py` |
| `over` / `under` / `invalid` | 'warn' / 'ignore' / 'warn' | 同上 |
| `bufsize`（ufunc 缓冲区） | 8192 字节 | `_ufunc_config.py:162` |
| `verify`（array_function_dispatch） | True（开发期校验签名） | `overrides.py:119` |
| `_NoValue` 单例身份 | 通过禁止 reload 保证 | `_globals.py` |

## 5. 错误与重试语义

- **签名不匹配**：`verify_matching_signatures` 检测 dispatcher 与实现函数签名不一致时抛出 `RuntimeError`（`overrides.py:110`）
- **dispatcher 默认值**：dispatcher 函数的默认值只能是 `None`（`overrides.py:114`）
- **like= 参数**：当 dispatcher=None 时，要求实现函数最后一个关键字-only 参数必须是 `like`（`overrides.py:168`）
- **无实现错误**：当所有参数的 `__array_function__` 都返回 NotImplemented 时，由 `array_function_errmsg_formatter` 格式化错误消息（`_internal.py:846`）

## 6. 并发细节

- `_ArrayFunctionDispatcher` 是 C 扩展类，分发逻辑在 C 层执行，不持有 Python 锁
- `ARRAY_FUNCTIONS` 是模块级 set，仅在 import 时填充，运行时只读
- `errstate` 是 Python 上下文管理器，通过 `seterr`/`geterr` 修改全局 ufunc 错误状态——**注意：这是进程级全局状态，非线程隔离**（与 `format_options` 的 ContextVar 不同）
- `_NoValue` 单例通过禁止模块 reload 保证身份稳定（`_globals.py`）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/overrides.py`：NEP-18 协议的 Python 包装
- `numpy/_core/_ufunc_config.py`：ufunc 错误/缓冲区配置
- `numpy/_globals.py`：全局单例

**Out-of-Scope（不在本仓库源码内）**

- `_ArrayFunctionDispatcher` C 类的实际分发逻辑——在 `_multiarray_umath` C 扩展中
- `_get_implementing_args` C 实现——同上
- `__array_ufunc__` 协议（NEP-17）的 C 层实现——在 ufunc 调用路径中
- 第三方数组库（torch/jax/sparse 等）——不在本仓库源码内

## 8. 与相邻子系统交互

- **上游**：所有 `@array_function_dispatch` 函数（numeric.py、fromnumeric.py、shape_base.py 等）都使用本叶子的装饰器
- **下游 → C 扩展**：`_ArrayFunctionDispatcher`、`_get_implementing_args` 从 `_multiarray_umath` 导入
- **下游 → 第三方**：`__array_function__` 协议将调用委托给第三方数组库

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **能力缝归组——覆盖协议（duck typing 能力缝）**：NEP-18 `__array_function__` 是 NumPy 最重要的 duck typing 扩展点。核心机制是装饰器模式：`@array_function_dispatch(dispatcher)` 将普通函数包装为 `_ArrayFunctionDispatcher`，在调用时先检查参数是否有覆盖方法。这使得第三方库（torch、jax、cupy 等）可以无缝接管 NumPy 函数调用
2. **注册表**：`ARRAY_FUNCTIONS` set 注册所有被包装的公开函数，供测试和文档生成使用
3. **配置驱动**：`verify=True` 是开发期检查开关，生产代码可关闭；`reduction` 参数启用 C 层精确 ndarray 快速路径
4. **哨兵单例**：`_NoValue` 通过禁止 reload 保证身份稳定，是区分"未传参"与"显式 None"的经典模式

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| overrides-ufunc-config 架构图 | `overrides-ufunc-config-architecture.html` | architecture | standard |

- JSON IR 源文件：`json/overrides-ufunc-config-architecture.json`
- 降档原因：showcase 校验因"包装"标签与 dispatcher 组件重叠而失败，调整 labelDy 后 standard 档通过
- 时序图：NEP-18 分发调用链已在第 3 节文字化描述；分发为线性单层检查，不适用额外时序图
