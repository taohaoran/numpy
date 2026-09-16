# ufunc 类型提升与分发（ufunc-dispatch）

> 本文是 `core-ufunc` 域下的叶子子系统文档。域级总览见 `../core-ufunc.md`。
> 本文聚焦 ufunc 的 dtype 提升与 ArrayMethod 匹配算法，不展开 ufunc 对象调用编排（见 `../ufunc-object/`）与内核实现（见 `../ufunc-loops/`）。
>
> 源码基准：numpy commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 添加 loop | `PyUFunc_AddLoop` 验证并写入 `ufunc->_loops` 字典 | `umath/dispatching.cpp:83` |
| 从 spec 注册 | `PyUFunc_AddLoopFromSpec_int` 从 ArrayMethod_Spec 创建并注册 | `umath/dispatching.cpp:145` |
| 批量注册 | `PyUFunc_AddLoopsFromSpecs` 从 LoopSlot 表批量注册 | `umath/dispatching.cpp:209` |
| 实现匹配 | `resolve_implementation_info` 遍历 `_loops`，找 dtype 超类匹配的最佳 loop | `umath/dispatching.cpp:390` |
| promoter 递归 | `call_promoter_and_recurse` 命中 promoter 时递归重新分发 | `umath/dispatching.cpp:667` |
| 旧式提升回退 | `legacy_promote_using_legacy_type_resolver` 兼容旧类型解析 | `umath/dispatching.cpp:801` |
| 提升+获取实现 | `promote_and_get_ufuncimpl` 完整分发主流程 | `umath/dispatching.cpp:1138` |
| 默认提升器 | `default_ufunc_promoter` 标准 dtype 提升规则 | `umath/dispatching.cpp:1259` |
| 逻辑 ufunc 提升 | `logical_ufunc_promoter` 逻辑运算专用提升 | `umath/dispatching.cpp:1357` |
| 对象 ufunc 提升 | `object_only_ufunc_promoter` 对象运算专用 | `umath/dispatching.cpp:1334` |
| 无匹配直接取 | `get_info_no_cast` 无转换直取实现 | `umath/dispatching.cpp:1448` |
| 注册 promoter | `PyUFunc_AddPromoter` 添加自定义提升器 | `umath/dispatching.cpp:1486` |
| 归约提升 | `reducelike_promote_and_resolve` 归约专用提升 | `umath/ufunc_object.c:2331,2476` |
| 循环工具头 | loops_utils.h.src：nomemoverlap 等辅助 | `umath/loops_utils.h.src` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `ufunc->_loops` | `ufuncobject.h:231` | OrderedDict：`DType 元组 → (DType 元组, ArrayMethod/promoter)` |
| `ufunc->_dispatch_cache` | `ufuncobject.h:229` | 分发结果缓存，避免重复提升 |
| `PyArrayMethod` | `array_method.h` | 算子实现对象：持有 get_strided_loop 回调与 flags |
| promoter | `dispatching.cpp:1486` | 提升回调：可修改 operand DType 后递归重新分发 |
| `resolve_implementation_info` | `dispatching.cpp:390` | 核心匹配函数：子类匹配 + 最佳选择 |
| `NPY_DT_is_abstract` | dtype 元类 | 判断 DType 是否为抽象基类 |

## 3. 关键调用链

### 3.1 分发主流程（五步，来自 dispatching.cpp 头注释）

1. **签名覆盖**：用户传入的 `signature`（dtype= 参数）覆盖 `operand_DTypes`
2. **查缓存**：检查 `_dispatch_cache`，命中则跳到步骤 5
3. **找最佳 loop**：`resolve_implementation_info` 遍历 `ufunc->_loops`：
   - 每个候选 loop 的注册 DType 必须是操作数 DType 的超类（或 None 通配）
   - 输出 dtype 未指定时始终匹配
   - 多候选时按"最具体匹配"原则选最佳（输入优先于输出）
4. **promoter 递归**：若命中的是 promoter 而非 ArrayMethod，调用 promoter 修改 DType 后回到步骤 2
5. **产出**：最终 ArrayMethod 的注册 dtype 复制到 signature，供 loop 执行使用

### 3.2 注册路径

1. 构建时经 `PyUFunc_AddLoopsFromSpecs`（:209）从 LoopSlot 表批量注册
2. 每个 spec 经 `PyUFunc_AddLoopFromSpec_int`（:145）转为 ArrayMethod 写入 `_loops`
3. 重复注册报错（除非 ignore_duplicate）

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `signature`/`dtype` | 固定输出 dtype，覆盖提升结果 | dispatching 步骤1 |
| `casting` | 转换规则：safe/same_kind/unsafe | `validate_casting` |
| `_dispatch_cache` | 自动缓存提升分发结果，无手动配置 | ufuncobject.h:229 |
| `ignore_duplicate` | AddLoop 时是否忽略重复注册 | dispatching.cpp:83 |

## 5. 错误与重试语义

- **无匹配 loop**：`resolve_implementation_info` 遍历完无匹配时返回未找到，调用方抛 TypeError
- **promoter 失败**：promoter 回调出错时直接上抛，不自动重试
- **重复注册**：相同 DType 元组重复注册且非 ignore_duplicate 时抛 TypeError
- **提升冲突**：多候选同等匹配时放弃解析（dispatching.cpp:498-510 注释）
- 无自动重试；promoter 递归有终止条件（返回 ArrayMethod 即终止）

## 6. 并发细节

- **GIL**：分发在持有 GIL 时执行（涉及 Python 对象字典操作）
- **缓存线程安全**：`_dispatch_cache` 经 OrderedDict 操作，Python GIL 保护
- **无内部锁**：分发本身无 mutex
- **promoter 递归**：可调用任意 Python 代码，可能触发副作用

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `umath/dispatching.cpp`：类型提升与分发核心算法
- `umath/dispatching.h`：接口声明
- `umath/loops_utils.h.src`：循环工具头
- `umath/legacy_array_method.c`：旧式 loop 适配

**Out-of-Scope（不在本仓库源码内）**
- 实际算子内核计算（见 `../ufunc-loops/`）
- ufunc 对象生命周期与调用编排（见 `../ufunc-object/`）
- DType 元类体系（归 dtype-system 子系统）
- 浮点错误处理（见 `../ufunc-error/`）

## 8. 与相邻子系统交互

- **上游 → ufunc-object**：对象层调用 `promote_and_get_ufuncimpl` 做提升分发
- **下游 → ufunc-loops**：选中 ArrayMethod 后经 `get_strided_loop` 获取内核
- **下游 → dtype-system**：依赖 DType 元类的子类关系判断
- **上游注册**：构建期经 `PyUFunc_AddLoopsFromSpecs` 批量注册内核

## 9. 语言专项适配口径

1. **注册表与工厂**：`_loops` 字典 = DType 元组 → ArrayMethod 注册表；`resolve_implementation_info` 是字符串/类路由工厂。
2. **promoter 链**：promoter 是可插拔提升回调，类似 transformers 的注册表钩子；可递归修改 DType 重新分发。
3. **双轨路径**：新版 ArrayMethod 路径为主，`legacy_promote_using_legacy_type_resolver` 兼容旧式 type_resolver 路径。
4. **依赖方向**：`dispatching.cpp` 依赖 `dtypemeta.h`（DType 元类）与 `array_method.h`（ArrayMethod），单向。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 架构图 | `ufunc-dispatch-architecture.html` | architecture | standard |

降档说明：含 promoter 递归回边与缓存分支，showcase 标签重叠校验未全过，降 standard 档。分发主流程为线性五步算法，已在 MD 第 3 节文字化描述，不再单独产出时序图。JSON IR 源文件位于 `json/` 目录。
