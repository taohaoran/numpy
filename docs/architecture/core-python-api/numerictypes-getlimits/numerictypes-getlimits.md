# 数值类型与机器限制（numerictypes-getlimits）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 类型层级定义 | generic → number → integer(signed/unsigned) / inexact(floating/complexfloating) / flexible(character/void) / object_ | `numpy/_core/numerictypes.py:42-76` |
| 类型注册表构建 | _type_aliases.py 从 C 层 typeinfo 构建 sctypeDict、allTypes、sctypes 三张注册表 | `numpy/_core/_type_aliases.py` |
| 别名映射 | double→float64、single→float32、int_→intp、float→float64 等 C 名/Python 名/比特宽名互映射 | `_type_aliases.py:51-95` |
| 类型查询函数 | issctype、obj2sctype、issubclass_、issubsctype、issubdtype、isdtype、sctype2char | `numerictypes.py:129,176,231,273,419,329,485` |
| 浮点机器限制 | finfo 类：float16/32/64/longdouble 的 eps、min、max、precision 等 | `numpy/_core/getlimits.py:52` |
| 整数机器限制 | iinfo 类：int8/16/32/64/uint 等的 min、max、bits 等 | `getlimits.py:342` |
| 自包含工具层 | numpy/_utils/：set_module 装饰器、_conversions（asbytes/asunicode）、_inspect（getargspec）、_pep440 | `numpy/_utils/__init__.py` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `sctypeDict` | `_type_aliases.py:27` | 名称→类型的字典，包含所有别名（"float"→float64 等） |
| `allTypes` | `_type_aliases.py:28` | 名称→类型的字典，仅包含正式名和 C 名 |
| `sctypes` | `_type_aliases.py:102` | 按类型组分组：int/uint/float/complex/others |
| `finfo` | `getlimits.py:52` | 浮点机器限制，cached_property 延迟计算 |
| `iinfo` | `getlimits.py:342` | 整数机器限制 |
| `issubdtype(arg1, arg2)` | `numerictypes.py:419` | 判断 dtype1 是否为 dtype2 的子类型 |
| `isdtype(dtype, kind)` | `numerictypes.py:329` | 判断 dtype 是否匹配 kind（如 'floating'、'integer'） |
| `set_module(module)` | `numpy/_utils/__init__.py:14` | 装饰器，覆盖函数/类的 `__module__` 属性 |

## 3. 关键调用链

### 3.1 类型注册表构建（import 时）

1. `_type_aliases.py` 被导入，从 `multiarray` 导入 `typeinfo`（C 层字典）
2. 遍历 `typeinfo.items()`：NPY_ 前缀键入 `c_names_dict`，其余入 `allTypes` 和 `sctypeDict`（`_type_aliases.py:41-49`）
3. 应用别名映射 `_aliases`（double→float64 等）和 `_extra_aliases`（float→float64 等）
4. 按 issubclass 关系将类型分组到 `sctypes` 的 int/uint/float/complex/others 集合
5. 每组按 itemsize 和名称排序

### 3.2 finfo 实例化

1. 用户 `np.finfo(np.float64)` → 进入 `getlimits.py:52` 的 `finfo.__init__`
2. `_finfo_get_realdtype(dtype)` 从 C 扩展获取实际浮点 dtype（复数取实部）
3. `_MACHAR_PARAMS` 查找对应的 itype 和格式串
4. 通过 `cached_property` 延迟计算 eps、min、max 等常量

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `_MACHAR_PARAMS` | double/single/longdouble/half 的 itype 和格式串映射 | `getlimits.py:32-48` |
| `sctypeDict` 别名 | 7 个别名（double/single/half/bool_/int_/uint + extra） | `_type_aliases.py:51-82` |

## 5. 错误与重试语义

- **issctype 失败**：无法识别的类型表示返回 False
- **finfo 非浮点类型**：对非浮点类型调用 finfo 抛出 ValueError
- **iinfo 非整数类型**：对非整数类型调用 iinfo 抛出 ValueError

## 6. 并发细节

- 本叶子为纯 Python 层，不管理线程/锁
- `sctypeDict`/`allTypes`/`sctypes` 在 import 时一次性构建，运行时只读
- `finfo`/`iinfo` 的 `cached_property` 是实例级缓存，多线程读取安全（Python 缓存写入非原子，但重复计算无害）

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/numerictypes.py`：类型层级和查询函数
- `numpy/_core/_type_aliases.py`：注册表构建
- `numpy/_core/getlimits.py`：finfo/iinfo
- `numpy/_utils/`：自包含工具层

**Out-of-Scope（不在本仓库源码内）**

- `typeinfo` C 扩展字典——在 `_multiarray_umath` 中
- 实际类型对象（int8/float64 等 C 类型对象）——在 `_multiarray_umath` 中
- `_finfo_get_realdtype`/`_populate_finfo_constants` C 函数——同上

## 8. 与相邻子系统交互

- **上游**：所有需要类型判断的模块（arrayprint、fromnumeric、_internal 等）都依赖本叶子的类型系统
- **下游 → C 扩展**：`_type_aliases.py` → `typeinfo`（C 层类型注册表）；`getlimits.py` → `_finfo_get_realdtype`
- **下游 → _utils**：所有模块使用 `set_module` 装饰器

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **注册表与工厂机制**：`sctypeDict` 是名称→类型的注册表，在 import 时从 C 层 `typeinfo` 构建。别名映射表（`_aliases`/`_extra_aliases`）是硬编码的兼容性映射，支持 `np.dtype("float")` 等模糊名称解析
2. **分类器模式**：`issubdtype`/`isdtype`/`issubsctype` 是类型分类查询函数，用于在运行时判断 dtype 类别
3. **延迟计算**：`finfo`/`iinfo` 使用 `cached_property` 延迟计算机器限制常量，避免实例化时的计算开销

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| numerictypes-getlimits 架构图 | `numerictypes-getlimits-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/numerictypes-getlimits-architecture.json`
- 降档原因：无，简化边后通过 showcase 校验
- 时序图/数据流图：注册表构建和 finfo 实例化为线性流程，不适用额外时序图
