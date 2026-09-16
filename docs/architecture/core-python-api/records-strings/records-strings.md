# 记录数组与字符数组（records-strings）

> 本文是 `core-python-api` 域下的叶子子系统文档。域级总览见 `../core-python-api.md`。
>
> 源码基准：NumPy main 分支，commit `3deeb20adb47f5da91e490ef8feda81114823710`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 记录数组 | record（void 子类，支持属性访问字段）、recarray（ndarray 子类，字段名可作属性） | `numpy/_core/records.py:196,279` |
| dtype 格式解析 | format_parser：将格式字符串/名称/标题转换为 dtype | `numpy/_core/records.py:57` |
| 记录数组构造 | fromarrays（从列数组）、fromrecords（从元组列表）、fromstring（从字符串）、fromfile（从文件） | `records.py:570,665,754,844` |
| 重复字段检测 | find_duplicate：检测字段名重复 | `records.py:47` |
| 字符数组类 | chararray（ndarray 子类，Numarray 遗留兼容）：向量化字符串操作 | `numpy/_core/defchararray.py:405` |
| 向量字符串函数 | equal/not_equal/greater/less/capitalize/center/count/decode/encode/endswith/find/join/lstrip/replace/split/startswith/strip/upper 等约 40 个 | `defchararray.py:57-400` |
| StringDType 再导出 | numpy/strings/ 从 _core.strings 再导出 StringDType 向量字符串函数 | `numpy/strings/__init__.py` |
| char 弃用薄层 | numpy/char/ 经 __getattr__ 转发到 defchararray，对 chararray/array/asarray 发 DeprecationWarning | `numpy/char/__init__.py` |
| rec 再导出薄层 | numpy/rec/ 从 _core.records 再导出 | `numpy/rec/__init__.py` |

## 2. 核心类型与接口清单

| 类型/函数 | 位置 | 职责 |
|-----------|------|------|
| `record(nt.void)` | `records.py:196` | 结构化标量，支持 `rec.field` 属性访问 |
| `recarray(ndarray)` | `records.py:279` | 结构化 ndarray，`self._row.field` 替代 `self['field']` |
| `format_parser` | `records.py:57` | 格式字符串→dtype 解析器，支持逗号分隔格式 |
| `chararray(ndarray)` | `defchararray.py:405` | 字符数组（Numarray 遗留），每个元素为定长字符串 |
| `array(obj, itemsize, ...)` | `defchararray.py:1221` | chararray 工厂函数 |
| `_binary_op_dispatcher(x1, x2)` | `defchararray.py:57` | NEP-18 dispatcher 辅助函数 |

## 3. 关键调用链

### 3.1 recarray 字段访问

1. 用户创建 `r = np.rec.array([(1, 2.0), (3, 4.0)], names='a,b')`
2. `rec.__getattribute__` 拦截属性访问：`r.a` → 查找 dtype 字段名 `'a'` → 返回该列视图
3. recarray 继承 ndarray，字段访问通过 `_field` 索引到结构化 dtype 的对应偏移

### 3.2 np.char.add 调用链

1. 用户调用 `np.char.add(a, b)` → 经 `__getattr__` 转发到 `defchararray.add`
2. 实际操作委托给 `numpy._core.strings` 的 C 扩展实现（StringDType 向量字符串函数）
3. 若访问 `np.char.chararray`，额外发出 DeprecationWarning（NumPy 2.5+）

## 4. 配置项

| 配置项 | 默认值/行为 | 位置 |
|--------|-------------|------|
| `itemsize`（chararray.array） | None（自动推断） | `defchararray.py:1221` |
| `unicode`（chararray.array） | None（自动推断 bytes_/str_） | 同上 |
| `byteorderconv` | 字节序字符映射表 | `records.py:23` |

## 5. 错误与重试语义

- **重复字段名**：`find_duplicate` 检测后由构造函数抛出 `ValueError`
- **chararray 弃用警告**：NumPy 2.5（2026-01-07）起，`numpy/char/` 对 `chararray`、`array`、`asarray` 发出 DeprecationWarning（`numpy/char/__init__.py`）
- **format_parser 格式错误**：无法解析的格式字符串抛出 ValueError

## 6. 并发细节

- 本叶子为 Python 包装层，不管理线程/锁
- 实际字符串操作在 `_core.strings` C 扩展中执行，释放 GIL

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `numpy/_core/records.py`：record/recarray/format_parser/构造函数
- `numpy/_core/defchararray.py`：chararray 类和向量字符串函数
- `numpy/strings/__init__.py`：StringDType 函数再导出
- `numpy/char/__init__.py`：char 弃用薄层
- `numpy/rec/__init__.py`：rec 再导出薄层

**Out-of-Scope（不在本仓库源码内）**

- `_core.strings` C 扩展（StringDType 实际实现）——在 C 扩展中
- `compare_chararrays` C 扩展函数——在 `_multiarray_umath` 中

## 8. 与相邻子系统交互

- **上游**：用户通过 `np.rec.*`、`np.char.*` 调用；`numpy/__init__.py` 再导出
- **下游 → C 扩展**：`defchararray.py` → `_core.strings`（StringDType 操作）、`compare_chararrays`
- **下游 → numeric**：`records.py` 使用 `numeric.ndarray`、`numerictypes`
- **相邻叶子**：`numerictypes-getlimits` 提供 dtype 类型系统

## 9. 语言专项适配口径

本叶子按「纯 Python 项目专项分析清单」分析：

1. **再导出薄层**：`numpy/char/` 和 `numpy/rec/` 都是再导出薄层。`numpy/char/` 额外使用 `__getattr__` + `__DEPRECATED` frozenset 实现选择性弃用警告——这是纯 Python 中"渐进式弃用"的典型模式
2. **遗留兼容模式**：chararray 是 Numarray 兼容遗留类，文档明确标注"not recommended for new development"，推荐使用 StringDType。这是 NumPy 从定长字符数组向 StringDType 迁移过程中的过渡层

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| records-strings 架构图 | `records-strings-architecture.html` | architecture | showcase |

- JSON IR 源文件：`json/records-strings-architecture.json`
- 降档原因：无
- 时序图/数据流图：字段访问和字符串操作为直接函数调用，不适用额外时序图
