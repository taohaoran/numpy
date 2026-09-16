# core-ndarray（核心数组对象域）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`numpy` git commit `3deeb20adb47f5da91e490ef8feda8111482371`，Python + C/Cython 混合，meson 构建。

## 1. 域职责

core-ndarray 域是 NumPy 的"数组对象核心"：定义并实现 ndarray 对象本体、其 dtype 描述符体系、任意 Python 对象到数组的强制（coercion）、数组的迭代与索引、datetime/timedelta 与新 StringDType。所有数值计算（ufunc 内核）在本域之外的 umath/ 域；本域负责"数组是什么、怎么构造、怎么遍历、怎么解释类型"。

核心代码路径：`numpy/_core/src/multiarray/`（C 扩展层 `_multiarray_umath`）+ `numpy/_core/_dtype.py`（Python 侧字符串化）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| ndarray-object | [ndarray-object.md](ndarray-object/ndarray-object.md) | [架构图](ndarray-object/ndarray-object-architecture.html) | — | ndarray 类型对象、结构体、创建/析构/协议/内存 |
| dtype-system | [dtype-system.md](dtype-system/dtype-system.md) | [架构图](dtype-system/dtype-system-architecture.html) | — | dtype 描述符、DType 元类、cast 安全表、类型提升 |
| conversion-coercion | [conversion-coercion.md](conversion-coercion/conversion-coercion.md) | [架构图](conversion-coercion/conversion-coercion-architecture.html) | — | 任意对象→数组/标量的强制与递归形状发现、标量类型 |
| iteration-indexing | [iteration-indexing.md](iteration-indexing/iteration-indexing.md) | [架构图](iteration-indexing/iteration-indexing-architecture.html) | — | 迭代器对象、序列协议、花哨索引 |
| datetime | [datetime.md](datetime/datetime.md) | [架构图](datetime/datetime-architecture.html) | — | datetime64/timedelta64、单位换算、工作日历 |
| stringdtype | [stringdtype.md](stringdtype/stringdtype.md) | [架构图](stringdtype/stringdtype-architecture.html) | — | 2.0 新 UTF-8 StringDType、SSO 存储、cast |

## 3. 域级机制细节

- **对象模型**：ndarray（`PyArrayObject_fields`）持有 `descr`（→dtype-system）、`base`（视图链压缩）、`mem_handler`（可插拔内存策略）。
- **生成 vs 手写边界**：`arraytypes.c.src`/`scalartypes.c.src` 是手写 C 模板，含 `@type@`/`@name@` 占位，经 meson `src_file.process()` 构建期按类型实例化；产物 `.c` 不入库。手写核心文件直接编译。
- **两层分发**：cast 安全表（编译期静态）+ dtype_transfer（运行期查表）；真正的算子 × 后端 ufunc 分发在 umath 域。
- **前端符号面**：C 扩展 `_multiarray_umath` 经 C-API 暴露给 Python；`_dtype.py` 仅处理 dtype 字符串化。
- **依赖方向**：`src/common` → multiarray 各子系统 → umath，单向。
- **GIL**：对象模型由 GIL 保护；大块 ufunc/排序执行时释放 GIL（gil_utils.h）。

## 4. 域级图

本域各叶子架构图见上表；系统级架构图见顶层 `../system-architecture.html`（组织者产出）。
