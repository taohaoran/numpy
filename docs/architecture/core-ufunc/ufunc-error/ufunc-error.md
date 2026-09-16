# ufunc 浮点错误处理（ufunc-error）

> 本文是 `core-ufunc` 域下的叶子子系统文档。域级总览见 `../core-ufunc.md`。
> 本文聚焦 ufunc 浮点异常的配置、存储与运行时处理，不展开 ufunc 调用编排（见 `../ufunc-object/`）。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 错误对象结构 | `npy_extobj`：bufsize、errmask 位掩码、pyfunc 回调 | `umath/extobj.h` |
| 初始化 | `init_extobj` 创建默认 capsule 与 ContextVar | `umath/extobj.c:144` |
| 构造 capsule | `make_extobj_capsule` 分配并封装 extobj 为 PyCapsule | `umath/extobj.c:88` |
| 取当前状态 | `fetch_curr_extobj_state` 从 ContextVar 取当前错误对象 | `umath/extobj.c:118` |
| 设置错误 | `extobj_make_extobj`（`_seterrobj`）解析参数生成新 capsule | `umath/extobj.c:207` |
| 查询错误 | `extobj_get_extobj_dict` 返回当前错误状态字典 | `umath/extobj.c:314` |
| 浮点错误处理 | `PyUFunc_handlefperr` 按 errmask 分发四类错误 | `umath/extobj.c:62` |
| 错误处理器 | `_error_handler` 六种模式：ignore/warn/raise/call/print/log | `umath/extobj.c:390` |
| 循环后检查 | `_check_ufunc_fperr` 读硬件 FPU 状态并处理 | `umath/extobj.c:538` |
| 公开 API | `PyUFunc_GiveFloatingpointErrors` 信号浮点错误 | `umath/extobj.c:512` |
| 提取参数 | `_get_bufsize_errmask` 取 bufsize 与 errormask | `umath/extobj.c:565` |
| Python 配置层 | `_ufunc_config.py`：seterr/errstate/geterr 封装 | `numpy/_core/_ufunc_config.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `npy_extobj` | `extobj.h` | 错误对象：`npy_intp bufsize`、`int errmask`、`PyObject *pyfunc` |
| `errmask` 位布局 | `extobj.c:30-38` | 四类错误各 3bit：divide(0-2)/over(3-5)/under(6-8)/invalid(9-11) |
| 错误模式常量 | `extobj.c:22-27` | IGNORE=0/WARN=1/RAISE=2/CALL=3/PRINT=4/LOG=5 |
| `PyContextVar` | Python C API | `npy_extobj_contextvar` 线程/协程隔离错误状态 |
| `_error_handler` | `extobj.c:390` | 按 method 分发到六种处理模式 |
| `npy_get_floatstatus_barrier` | `npy_fenv.c` | 读取硬件 FPU 异常状态寄存器 |

## 3. 关键调用链

### 3.1 设置错误模式（np.seterr）

1. Python `np.seterr(divide='raise', ...)` → `_ufunc_config.py` 调用 `_seterrobj`
2. `extobj_make_extobj`（:207）解析 all/divide/over/under/invalid 与 bufsize/call
3. `fetch_curr_extobj_state`（:118）取当前 ContextVar 中的 capsule
4. 按新模式修改 errmask 位段，`make_extobj_capsule`（:88）生成新 capsule
5. 新 capsule 赋给 ContextVar，对调用方上下文生效

### 3.2 循环后错误检查

1. `execute_ufunc_loop`（ufunc_object.c:1232）循环结束后调用 `_check_ufunc_fperr(errormask, name)`
2. `_check_ufunc_fperr`（:538）：
   - `npy_get_floatstatus_barrier` 读硬件 FPU 状态
   - 无错误返回 0
   - `_extract_pyvals` 取 errmask 与 pyfunc
   - `PyUFunc_handlefperr`（:62）按 HANDLEIT 宏逐类检查
3. `_error_handler`（:390）按 mode 处理：
   - IGNORE：直接返回
   - WARN：`PyErr_WarnEx(RuntimeWarning)`
   - RAISE：`PyErr_Format(FloatingPointError)`
   - CALL：调用用户 pyfunc
   - PRINT：fprintf stderr
   - LOG：调用 pyfunc.write

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| divide 模式 | 默认 warn | `extobj.c:41-44` |
| over 模式 | 默认 warn | 同上 |
| under 模式 | 默认 ignore | 同上 |
| invalid 模式 | 默认 warn | 同上 |
| bufsize | 默认 `NPY_BUFSIZE`，需 5..10e6 且 16 倍数 | `extobj.c:231-249` |
| callable | CALL/LOG 模式的 Python 回调对象 | `extobj.c:252` |
| `np.errstate` | 上下文管理器临时修改 | `_ufunc_config.py` |

## 5. 错误与重试语义

- **raise 模式**：设置 `PyExc_FloatingPointError`，循环返回 -1 上抛
- **warn 模式**：发 RuntimeWarning，循环继续
- **call 模式**：调用用户回调，回调失败则上抛
- **无重试**：浮点错误不重试，按配置处理后继续或中止
- **ContextVar 隔离**：不同线程/协程可有不同错误配置

## 6. 并发细节

- **ContextVar 隔离**：错误状态经 `PyContextVar` 存储，线程/协程独立
- **GIL 释放**：循环执行时释放 GIL；错误处理（_error_handler）需 C API，经 `NPY_ALLOW_C_API`/`NPY_DISABLE_C_API` 包裹
- **immortal capsule**：默认 capsule 在 free-threaded Python 下设为 immortal（`extobj.c:152-158`）
- **无内部锁**：每次调用复制 extobj 副本，无共享可变状态

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `umath/extobj.c`：错误对象创建、设置、查询、处理
- `umath/extobj.h`：npy_extobj 结构声明
- `numpy/_core/_ufunc_config.py`：seterr/errstate Python 封装

**Out-of-Scope（不在本仓库源码内）**
- 硬件 FPU 状态读取（`common/npy_fenv.c`，归平台层）
- ufunc 循环执行本身（见 `../ufunc-object/`、`../ufunc-loops/`）

## 8. 与相邻子系统交互

- **上游 → ufunc-object**：循环结束后 `_check_ufunc_fperr` 被调用
- **上游 → Python 配置**：`np.seterr`/`np.errstate` 经 `_seterrobj` 修改 ContextVar
- **下游 → npy_fenv**：调用 `npy_get_floatstatus_barrier` 读硬件状态
- **下游 → Python 回调**：CALL/LOG 模式调用用户 Python 函数

## 9. 语言专项适配口径

1. **绑定边界**：C 层 `npy_extobj` 经 PyCapsule 暴露给 Python；`_ufunc_config.py` 纯做 seterr/errstate 封装。
2. **线程隔离**：用 Python `ContextVar` 而非 TLS 实现协程安全的错误状态隔离。
3. **GIL 管理**：循环释放 GIL 计算，错误处理需要 C API 时用 `NPY_ALLOW_C_API` 临时获取。
4. **依赖方向**：`extobj.c` 依赖 `npy_fenv.c`（硬件状态读取）与 Python C API。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `ufunc-error-architecture.html` | architecture | standard |

降档说明：含 ContextVar 存储与运行时处理两层 7 组件 6 连线，showcase 标签重叠校验未全过，降 standard 档。错误处理为线性分发流程，已在 MD 第 3 节文字化描述。JSON IR 源文件位于 `json/` 目录。
